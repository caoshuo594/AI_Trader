# AI_Trader_Pro 使用说明

## 1. 先看这里：怎么用

### 1.1 安装

1. 将 `AI_Trader_Pro.ex5` 放到当前 MT5 终端的 `MQL5\Experts\AI_Trader_Pro\` 目录。
2. 在 MT5 导航器中刷新 EA 列表。
3. 把 EA 拖到目标图表。
4. 打开 MT5 顶部 `Algo Trading`。
5. 在 `工具 -> 选项 -> EA交易` 中允许算法交易。

说明：

- 公开仓库默认只提供编译文件 `AI_Trader_Pro.ex5` 和文档，不提供 `.mq5` 源文件。

### 1.2 WebRequest 白名单

在 MT5 的 `工具 -> 选项 -> EA交易 -> WebRequest` 中加入：

- `InpLicenseLoginEndpoint` 对应域名，例如 `https://caocaoshuoshuo.com`
- `InpOpenRouterEndpoint`，默认接口为 `https://openrouter.ai/api/v1/chat/completions`

如果没有加入白名单，授权登录和模型调用都会失败。

### 1.3 首次启动需要填什么

首次使用时至少需要配置：

- `InpLicenseLoginEndpoint`
- `InpLicenseEmail`
- `InpLicensePassword`
- `InpOpenRouterApiKey`

如果 `InpRememberApiKey=true`，EA 会把 API Key 保存到本地，后续重启时可自动读取。

### 1.4 启动后的实际流程

当前版本的启动顺序是：

1. 先等待 MT5 已连接交易服务器且账号登录成功。
2. 校验本地授权；如果本地没有有效授权，则要求输入邮箱和密码联网登录一次。
3. 首次满足条件后立即执行一次分析。
4. 之后只在 `InpAnalysisTimeframe` 出现新 K 线时再次分析。

补充说明：

- `InpTimerSeconds` 只是轮询检查频率，不是实际交易频率。
- EA 初始化时会自动把当前图表切换到 `InpAnalysisTimeframe`。
- 若 `TERMINAL_CONNECTED` 状态刷新滞后，但账号、服务器名或服务器时间已经可读，当前版本会视为会话已就绪，不会一直卡在“等待连接”。

### 1.5 一轮分析会发生什么

每一轮的实际行为如下：

1. 抓取当前图表截图。
2. 生成结构化行情摘要 `market_context`。
3. 把“截图 + 行情摘要”一起发给 3 个专家模型。
4. 只有当 3 个模型严格共识，且本地风控通过时，EA 才会下单。

重要限制：

- 当前版本强制要求截图可用，不允许退化成纯文本分析。
- 如果截图重试后仍失败，本轮会直接结束，不调用 AI。
- 当前版本没有第二轮模型复核。
- 当前版本不会把交易或信号自动上传到外部后端，审计仅保存在本地。

### 1.6 最小排查清单

如果 EA 没有分析或没有下单，先检查这几项：

- MT5 是否已连接交易服务器
- 账号是否已登录
- `Algo Trading` 是否开启
- `WebRequest` 白名单是否配置正确
- 本地授权是否存在且未过期
- `InpOpenRouterApiKey` 是否可用
- 当前分析周期是否已经出现新 K 线
- 图表截图是否成功

## 2. 这个版本具体做什么

`AI_Trader_Pro` 是一个运行在 MT5 上的 AI 共识交易 EA。当前版本的核心特点是：

- 用 3 个独立专家模型做同一轮判断
- 强制依赖图表截图，不做纯文本降级
- 只有三模型严格同向且置信度达标时才允许交易
- 下单前仍要经过本地风控过滤
- 所有审计和调用记录只保存在本地

## 3. 当前支持的模型方式

默认有 3 个专家槽位：

- 专家1：`InpExpert1ModelPreset` / `InpExpert1CustomModel`
- 专家2：`InpExpert2ModelPreset` / `InpExpert2CustomModel`
- 专家3：`InpExpert3ModelPreset` / `InpExpert3CustomModel`

预设白名单模型：

- `google/gemini-3.1-pro-preview`
- `google/gemini-3-flash-preview`
- `openai/gpt-5.4`
- `openai/gpt-5.4-mini`
- `anthropic/claude-opus-4.6`
- `anthropic/claude-sonnet-4.6`
- `moonshotai/kimi-k2.5`

说明：

- 前 3 个专家默认走 `InpOpenRouterEndpoint` + `InpOpenRouterApiKey`。
- 如果某个专家槽位选择 `CUSTOM`，必须填写对应的 `InpExpertXCustomModel`。

## 4. 单轮分析输入

每个模型在当前版本只会看到这一种输入：

1. 图表截图 + 结构化行情摘要

结构化行情摘要 `market_context` 当前包含：

- `symbol`
- `timeframe`
- `server_time`
- `session`
- `spread_points`
- `atr14_points`
- `recent_range_points`
- `avg_close_move_points`
- `event_window`
- 最近 5 根 K 线的 `OHLC`

截图失败时的行为：

- 当前轮内部会先执行截图重试。
- 如果重试后仍失败，本轮直接结束，不调用任何 AI 模型。
- 如果截图成功但读取 Base64 失败，本轮同样直接结束，不会退化到文本分析。

## 5. 模型返回格式

当前版本要求模型返回单个 JSON，对应字段为：

- `action`: `BUY | SELL | HOLD`
- `confidence`: `0.0 ~ 1.0`
- `reason`
- `invalidation`
- `holding_horizon`: `scalp | intraday | swing`
- `risk_level`: `LOW | MEDIUM | HIGH`
- `setup_type`: `trend | reversal | breakout | range | noise`
- `reason_codes`

字段解读：

- `confidence`
  表示模型对本次 `action` 判断的主观把握程度，范围是 `0.0 ~ 1.0`。
  当前版本里，`confidence` 会直接参与是否允许下单的判断；只有当 `BUY/SELL` 且 `confidence >= InpMinConfidence` 时，才算有效支持票。

- `risk_level`
  表示模型对这次交易方案风险高低的定性标签：
  `LOW` = 相对稳健，`MEDIUM` = 中等风险，`HIGH` = 风险偏高。
  当前版本里，`risk_level` 主要用于解释、审计和日志展示，不会单独触发禁止下单。

如果模型返回为空、被截断、HTTP 失败、解析失败，都会被视为该模型本轮无效。

## 6. 共识逻辑

当前版本采用严格的三模型一致规则：

1. 只统计调用成功且解析成功的模型，记为“有效模型”。
2. 必须有 3 个有效模型，否则直接放弃。
3. 只把 `action=BUY/SELL` 且 `confidence >= InpMinConfidence` 的结果计为“合格支持票”。
4. 只有当 3 个模型都给出 `BUY`，且 3 个都满足最低置信度时，才允许做多。
5. 只有当 3 个模型都给出 `SELL`，且 3 个都满足最低置信度时，才允许做空。
6. 只要有 1 个模型不同意、返回 `HOLD`、置信度不足，或调用失败，就不下单。

当前常见拒绝原因：

- `INSUFFICIENT_VALID_MODELS`
- `ROUND1_HOLD`
- `ROUND1_DISAGREEMENT`
- `LOW_CONFIDENCE`

## 7. 本地风控与停机

模型共识成立后，还要通过本地风控：

- 点差过滤：`InpMaxSpreadPoints`
- 事件窗口过滤：`InpEnableNewsFilter`、`InpNewsBlockMinutesBefore`、`InpNewsBlockMinutesAfter`，当前只拦截 `HIGH` 等级事件
- 单品种单仓限制：`InpAllowOnePositionOnly`
- 下单预检：`OrderCheck`
- 连续亏损停机：`InpMaxConsecutiveLosses` + `InpLossCooldownMinutes`
- 日内最大回撤停机：`InpDailyMaxDrawdownPercent`

停机规则说明：

- 连续亏损达到阈值后，会暂停到 `InpLossCooldownMinutes` 指定的分钟数结束。
- 日内回撤达到阈值后，会停机到下一交易日。

## 8. 仓位与止盈止损

仓位模式：

- `InpVolumeMode = VOLUME_MODE_RISK_PERCENT`：按 `InpRiskPercent`、账户权益和止损点数计算手数。
- `InpVolumeMode = VOLUME_MODE_FIXED_LOTS`：直接使用 `InpFixedLots`。

保护性价格：

- `InpStopLossMode` 支持固定点数或 ATR 倍数。
- `InpTakeProfitMode` 支持固定点数或 ATR 倍数。
- ATR 周期使用 `InpAtrPeriod`。
- ATR 倍数分别使用 `InpStopLossAtrMultiplier`、`InpTakeProfitAtrMultiplier`。

## 9. 截图与本地文件

截图目录默认是：

- `MQL5\Files\Experts\AI_Trader_Pro`

相关参数：

- `InpScreenshotFolder`
- `InpKeepScreenshots`
- `InpKeepAuditScreenshots`
- `InpImageWidth`
- `InpImageHeight`
- `InpImageFormat`

删除规则：

- 只有当 `InpKeepScreenshots=false` 且 `InpKeepAuditScreenshots=false` 时，截图才会在本轮结束后删除。
- 只要其中一个为 `true`，截图就会保留。

## 10. 日志与信号审计

图表内日志通过 `InpShowCommentLog` 控制：

- `true`：显示左上角 `Comment()` 文本
- `false`：立即清空并关闭左上角日志显示

如果启用 `InpEnableSignalAudit=true`，每一轮都会写入 `InpSignalAuditFile`。当前审计内容包括：

- `cycle_id`
- `account_number`
- `symbol`
- `timeframe`
- `created_at`
- `screenshot_path`
- `screenshot_failure_reason`
- `analysis_mode`
- `analysis_executed`
- `analysis_skip_reason`
- `market_context`
- `final_action`
- `final_reason`
- `trade_placed`
- `analysis_round`
- `first_round`
- `risk_review`
- `model_audits`

其中：

- `analysis_round` / `first_round` 保存当前 3 个专家槽位的结构化结果。
- `risk_review` 当前固定为空数组，仅为兼容保留。
- `model_audits` 会记录每次模型调用的 `prompt`、请求体、响应体、响应头、HTTP 状态、错误码、延迟、摘要等详细信息。
- 可以直接用 `analysis_executed=true/false` 判断这轮是否真的发生了带图 AI 分析。
- 当前版本固定要求 `analysis_mode="VISION_REQUIRED"`；如果截图不可用，通常会看到 `analysis_executed=false` 且 `analysis_skip_reason="SCREENSHOT_UNAVAILABLE"`。

说明：

- 当前版本只做本地审计。
- 审计文件和截图都属于本地敏感数据，不适合直接公开。

## 11. API Key 保存机制

- 首次可直接在参数里填写 `InpOpenRouterApiKey`。
- 若 `InpRememberApiKey=true`，EA 会把 key 保存到当前终端实例的 `MQL5\Files` 下。
- 默认文件名为 `AI_Trader_Pro.key`，可通过 `InpApiKeyStoreFile` 修改。
- 下次启动如果参数为空，会优先尝试从本地文件读取。
- 如果旧版本的 key 在 `Terminal\Common\Files`，当前版本会尝试读取并迁移到当前终端目录。

注意：

- 这里是本地轻度混淆保存，不是强安全保险箱。
- MT5 参数输入框本身不支持密码星号显示。

## 12. EA 授权机制

- 首次使用时，需要填写：
  - `InpLicenseLoginEndpoint`
  - `InpLicenseEmail`
  - `InpLicensePassword`
- EA 会调用后端登录接口验证账号，并读取当前授权到期时间。
- 登录成功后，EA 会把授权信息缓存到当前终端的 `MQL5\Files\Experts\AI_Trader_Pro\` 目录。
- 只要本地授权未过期，后续启动不需要再次联网登录。
- 如果本地授权已过期，用户在网站续费后，需要重新在 EA 中输入邮箱和密码登录一次。
- 若 `expires_at` 为空，EA 视为永久授权。

注意：

- 当前方案不做 MT5 账号数量限制。
- 当前方案以本地缓存为准，不会在每次启动时回源校验。
- 如果需要切换到另一个账号，可删除本地授权缓存文件后重新登录。

## 13. 常见问题

### Q1: 为什么 EA 不启动分析

常见原因：

- 本地授权不存在且没有填写邮箱密码
- 本地授权已过期，需要续费后重新登录
- `InpLicenseLoginEndpoint` 没有加入 `WebRequest` 白名单
- 邮箱或密码错误
- MT5 还没连上交易服务器
- 账号还没登录成功
- 图表 K 线数据还没准备好
- 当前周期还没有新 K 线
- `InpOpenRouterApiKey` 为空且本地也没读到 key

### Q2: 为什么没下单

常见原因：

- 有效模型少于 3 个
- 任意一个模型返回 `HOLD`
- 3 个模型没有全部同方向一致
- 任意一个模型的 `confidence` 未达到 `InpMinConfidence`
- 点差超限
- 落入高等级事件窗口
- 连续亏损停机
- 日内回撤停机
- 已有持仓且 `InpAllowOnePositionOnly=true`
- `OrderCheck` 预检失败
- 自动交易权限或 WebRequest 配置不正确

### Q3: 为什么看不到截图

- 到 `MQL5\Files\Experts\AI_Trader_Pro` 或你自定义的 `InpScreenshotFolder` 下查看
- 如果两个保留参数都为 `false`，截图会在该轮结束后删除

### Q4: 本地审计文件在哪里

- 默认是 `MQL5\Files\Experts\AI_Trader_Pro\signal_audit.jsonl`
- 也可以通过 `InpSignalAuditFile` 改成别的相对路径

### Q5: 网站公开摘要怎么生成

当前 EA 不会自动上传数据。若你要更新 `website/` 下的公开展示数据，可在本地手动运行：

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\sync-audit-data.ps1
node .\website\scripts\build-public-summary.mjs .\website\data\trade-history.json .\website\data\public-summary.json
```

## 14. 当前默认参数参考

- `InpAnalysisTimeframe = PERIOD_M15`
- `InpTimerSeconds = 10`
- `InpImageWidth = 1920`
- `InpImageHeight = 1080`
- `InpImageFormat = "png"`
- `InpVolumeMode = VOLUME_MODE_RISK_PERCENT`
- `InpRiskPercent = 1.0`
- `InpFixedLots = 0.1`
- `InpStopLossMode = PROTECTIVE_LEVEL_FIXED_POINTS`
- `InpStopLossPoints = 300`
- `InpTakeProfitMode = PROTECTIVE_LEVEL_FIXED_POINTS`
- `InpTakeProfitPoints = 600`
- `InpAtrPeriod = 14`
- `InpMinConfidence = 0.60`
- `InpMaxSpreadPoints = 80`
- `InpNewsBlockMinutesBefore = 15`
- `InpNewsBlockMinutesAfter = 15`
- `InpMaxConsecutiveLosses = 5`
- `InpLossCooldownMinutes = 120`
- `InpDailyMaxDrawdownPercent = 3.0`
- `InpGeminiVisionMaxTokens = 4096`
- `InpGeminiTextMaxTokens = 2048`

建议先在模拟盘验证一段时间，再调整风险参数和模型组合。
