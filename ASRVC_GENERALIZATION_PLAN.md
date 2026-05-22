# ASRVC v2.1 跨资产原版内核与概率系统重构方案

## Summary

本方案重新定义 `ASRVC.pine` 的优化方向：不再把 ASRVC 简化成“EMA/LinReg + ATR 通道”，而是回到 ASR-VC 的核心逻辑：量价 K 线、平均支撑压力彩虹通道、市场结构/概率分析、参考信号和警报。

根据 TradingView 对 ASR-VC v1.5 的公开说明，ASR 通道本质上不是普通均线，而是基于一段时间内高低价平均值、波动率系数和多层通道权重生成的平均支撑压力系统。当前本地 `Backup_ASRVC.pine` 是实现参考，TradingView 页面和 CryptoPainter 的公开推文/镜像内容作为产品方向参考。

参考来源：

- [TradingView: CryptoPainter ASR-VC v1.5](https://cn.tradingview.com/script/0uDF7Kke/)
- [CryptoPainter X profile](https://x.com/CryptoPainter)
- [Followin mirror: ASR-VC trend/range and A-Trading comments](https://followin.io/en/feed/23034936)
- [Followin mirror: ASR-VC 15m oversold / spot premium state update](https://followin.io/en/feed/18287345)
- [Followin mirror: ASR-VC 4h channel state update](https://followin.io/feed/18347685)

## Key Research Takeaways

- ASR-VC 的公开定义是 `Average Support and Resistance + Volume Candles`，核心视觉是缓慢移动的彩虹支撑压力通道，而不是普通均线通道。
- TradingView 页面明确提到，通道来自最近一段时间 K 线高低价平均值，再经过波动率系数加权，形成多重支撑压力边沿。
- ASR-VC 更适配 4 倍基数周期，例如 `1m / 4m / 15m / 1h / 4h`；其他周期可以使用，但视觉标准度可能下降。
- 原版默认最适合 BTCUSD，后续版本增加 ETH 参数组，并在 v1.35 将 BTC 参数组设为通用参数组。
- v1.5 的关键新增是实时概率分析系统：用多个趋势/震荡指标作为数据源，按权重汇总每根 K 线的上涨/下跌概率，并区分趋势模式与震荡模式。
- CryptoPainter 的推文/镜像内容强调：中轨、通道状态、现货溢价、趋势/震荡结构、不要主观追单，以及用多周期策略筛选/进化来适配市场状态。

## Implementation Direction

- 主文件继续以 `Backup_ASRVC.pine` 的原版结构为基底，保留大量原版 ratio、逐线平滑、填色、表格和信号结构。
- 当前 `ASRVC_v2_compact_experiment.pine` 只保留为实验参考，不再作为主线。
- 新增真正的 ASR 通道内核层：
  - `avgHigh = ta.sma(high, length)`
  - `avgLow = ta.sma(low, length)`
  - `asrMid = (avgHigh + avgLow) / 2`
  - `asrRange = avgHigh - avgLow`
  - 再叠加 ATR rank、真实波幅、通道斜率和资产 profile 宽度系数。
- Legacy BTC/ETH 继续保留原版硬编码分支，用于对照旧版效果。
- Auto / 非加密资产默认走 `ASR General Kernel`，不再简单复用 BTC ratio 后做缩放。
- 修正周期分支顺序，避免 `timeframe_minutes >= 240` 抢先吞掉 `1D / 1W` 等特殊分支。
- 保留 2500 行以上的大结构，但删除或隔离明显重复、无效、不可达或会干扰编译的代码。代码量要服务于原版保真和可维护性，而不是堆死代码。

## Cross-Asset Profiles

资产配置继续保留：

- `Auto`
- `Crypto`
- `Equity/Index`
- `FX`
- `Bond/Rates`
- `Futures/CFD`
- `Fund`
- `ASRVC Original BTC`
- `ASRVC Original ETH`

Auto 使用 `syminfo.type` 识别：

- `crypto` -> Crypto
- `stock/index` -> Equity/Index
- `forex` -> FX
- `bond` -> Bond/Rates
- `futures/cfd` -> Futures/CFD
- `fund` -> Fund
- unknown -> Equity/Index

每个 profile 至少定义：

- ASR 长度倍率
- 通道宽度倍率
- 平滑强度
- 成交量权重
- 概率系统权重
- 是否启用 crypto-only 数据，例如现货溢价

期权继续完全排除。

## Probability System

新增 v1.5 风格概率系统，分为两个模式：

- `Range Mode`
- `Trend Mode`

概率系统输入源：

- ASR 通道位置：价格处于蓝区、白区、橙区、外轨的位置。
- ASR 通道斜率：通道整体上行、走平、下行。
- 中轨状态：站上、跌破、反复穿越。
- ADX/DMI：趋势强度和方向。
- ATR Rank：当前波动率位置。
- Compression Score：压缩/扩张状态。
- Volume Pressure：放量方向，FX/债券等自动降权。
- Momentum：短期动量。
- Crypto Premium：仅 Crypto 可选启用，默认非 Crypto 关闭。

输出：

- `Up Probability`
- `Down Probability`
- `Dominant Bias`
- `Probability Mode`
- `Confidence`

表格显示概率，但文案保持参考性质，不直接写成买卖指令。

## Market State Machine

状态机升级为稳定、不闪烁的结构：

- `Trend Up`
- `Trend Down`
- `Range Bullish`
- `Range Bearish`
- `Compression`
- `High Volatility`
- `Demand Shock`
- `Supply Shock`

状态切换使用 hysteresis，避免一两根 K 线导致反复变色。

状态机影响：

- 中轨颜色
- 区域填色
- 信号文案
- 概率系统权重
- 警报条件
- 平滑速度

## Signals And Alerts

保留 ASR-VC 的“参考信号”定位，不把信号描述成确定性交易建议。

警报分组：

- 状态切换
- 中轨站回/跌破
- 外轨触及
- 橙区/蓝区放量反应
- 压缩突破
- 趋势确认
- 概率翻转
- Long/Short reference levels

参考信号命名建议：

- `EnterLong1 / EnterLong2 / EnterLong3`
- `EnterShort1 / EnterShort2 / EnterShort3`
- `TrendLong / TrendShort`
- `LongTP / ShortTP`
- `AllLongExit / AllShortExit`

不实现自动交易，不改为 `strategy()`。

## Visual And Performance

- 保留彩虹通道、区域填色、量价 K 线、底部信息面板。
- 新增 `Clean Mode`：
  - 关闭背景填色
  - 关闭概率数字逐 K 绘制
  - 关闭部分标签
  - 保留主通道和表格
- 表格分为两档：
  - `Compact`: 状态、位置、概率、信心、成交量。
  - `Full`: 加入 profile、vol regime、alert status、support/resistance 距离。
- 对标签、line、box 数量做上限控制，避免 v1.5 页面提到的浏览器缓存/性能压力。
- 默认展示 5 根关键通道数值，避免数据栏过载。

## Test Plan

TradingView 编译验证：

- Pine v6 无编译错误。
- 无 `series int` 传给要求 `simple int` 的函数。
- 无数组越界。
- 无 `table.clear` 范围错误。
- 无 `na` / 除零导致的异常表格。

视觉验证：

- `FLY` 5m：曲线比 compact v2 更平滑，不能出现明显折角、台阶、突然压缩。
- `BTCUSD / BTCUSDT.P`: 15m、1h、4h、1D。
- `ETHUSD / ETHUSDT`: 15m、1h、4h、1D。
- `SPY / QQQ / AAPL`: 15m、1D。
- `TLT / IEF`: 1D。
- `EURUSD`: 15m、4h。
- `ES1! / NQ1!`: 15m、1D。
- `GC1! / XAUUSD`: 1h、1D。
- `VTI / GLD`: 1D。

验收标准：

- `ASRVC.pine` 保持原版大结构，目标 2500 行以上。
- BTC/ETH Legacy 模式视觉尽量接近原版。
- Auto 模式在非加密资产上不需要手动切 BTC/ETH。
- 通道顺序稳定：上轨 > 中轨 > 下轨。
- 低成交量或成交量不可靠资产不会出现错误放量信号。
- 概率系统可以在趋势/震荡模式之间表现出不同权重。
- 图表性能可接受，Clean Mode 能明显降低负担。

## Assumptions

- 本项目是个人研究和自用脚本，不复制或声称复刻 invite-only 源码。
- TradingView 页面是公开功能说明，`Backup_ASRVC.pine` 是本地实现参考。
- X 页面部分内容无法稳定直接抓取，使用公开 X 链接和 Followin 镜像内容辅助理解。
- 期权不进入本方案。
- 继续使用 `indicator()`，不改为 `strategy()`。
