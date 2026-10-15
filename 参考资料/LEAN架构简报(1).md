# QuantConnect LEAN 架构简报

给其他 AI 的事实底稿。依据公开仓库 `QuantConnect/Lean` 的 `Engine` 与 `Launcher/config.json`。类名以该仓库为准。

策略只实现 `QCAlgorithm`。行情、撮合、时钟、结果是可替换 Handler。`Launcher/config.json` 先加载顶层键，再用 `environment` 覆盖。换环境即可在回测、模拟盘、实盘之间切换，策略类不变。

## 分层

| 层 | 类型 | 职责 |
| --- | --- | --- |
| 策略 | `QCAlgorithm`，或 Algorithm Framework 五个模型 | 订阅、信号、下单请求 |
| 入口 | `Launcher`（`Program.cs`），或 Lean CLI + Docker | 读配置、取任务、`Engine.Run` |
| 引擎 | `QuantConnect.Lean.Engine.Engine`、`AlgorithmManager` | 按时间片驱动 Handler |
| 领域 | `Security`、`Slice`、`Order`、`Portfolio`、`Universe` | 循环里流转的对象 |
| 外部 | 本地数据 / `IDataQueueHandler`；`IBrokerageModel`；`IBrokerage` | 文件或实时流；规则；券商连接 |

仓库目录：`Algorithm`（基类与回调）、`Algorithm.Framework`（五段模型）、`Algorithm.Python`（Python 装载）、`Common`（领域类型）、`Engine`（循环与 Handler）、`Brokerages`（连接）、`Indicators`（指标与 Consolidator）、`Launcher`（入口与 `config.json`）。

## 进程如何接上

1. `Launcher` 合并命令行与 `config.json`，用 `environment` 覆盖顶层。
2. `LeanEngineSystemHandlers`：任务队列、API、消息、`ILeanManager`。
3. `LeanEngineAlgorithmHandlers`：装配、行情、订单、结果、时钟、历史。
4. `JobQueue.NextJob` 取出算法包。
5. 创建 `AlgorithmManager`，`ILeanManager.Initialize`，然后 `Engine.Run(job, algorithmManager, assemblyPath, workerThread)`。
6. `Engine` 创建算法实例与 `DataManager` / `UniverseSelection`，在 `Initialize` 之前初始化 `IDataFeed` 和 `IObjectStore`，以便订阅能被数据层截获。
7. `ISetupHandler` 设置资金、起止时间、经纪商模型并调用 `Initialize`。
8. `AlgorithmManager.Run` 进入时间片循环。用户代码跑在隔离器里。

## 接口与实现

短名。完整前缀见下一节。

| 接口 | 职责 | `backtesting` | `live-paper` | `live-interactive` |
| --- | --- | --- | --- | --- |
| `ISetupHandler` | 创建算法、设现金与模型、调用 `Initialize` | `BacktestingSetupHandler` | `BrokerageSetupHandler` | `BrokerageSetupHandler` |
| `IDataFeed` | 订阅变成 `Slice` | `FileSystemDataFeed` | `LiveTradingDataFeed` | `LiveTradingDataFeed` |
| `IDataQueueHandler` | 实盘原始流 | 无，读磁盘 | `LiveDataQueue` | `InteractiveBrokersBrokerage` |
| `ITransactionHandler` | 订单如何成交 | `BacktestingTransactionHandler` | `BacktestingTransactionHandler` | `BrokerageTransactionHandler` |
| `IResultHandler` | 日志、图、权益曲线 | `BacktestingResultHandler` | `LiveTradingResultHandler` | `LiveTradingResultHandler` |
| `IRealTimeHandler` | `schedule` 与日终 | `BacktestingRealTimeHandler` | `LiveTradingRealTimeHandler` | `LiveTradingRealTimeHandler` |
| `IHistoryProvider` | `History()` | `SubscriptionDataReaderHistoryProvider` | 不覆盖，沿用上层 | `BrokerageHistoryProvider` + `SubscriptionDataReaderHistoryProvider` |
| `IBrokerage` | 账户与报单连接 | 无 | `PaperBrokerage` | `InteractiveBrokersBrokerage` |

`config.json` 键：`setup-handler`、`data-feed-handler`、`data-queue-handler`、`transaction-handler`、`result-handler`、`real-time-handler`、`history-provider`、`live-mode`、`live-mode-brokerage`。

系统侧接口不参与撮合：`IJobQueueHandler`、`IApi`、`IMessagingHandler`、`ILeanManager`。数据辅助：`IDataProvider`（`DefaultDataProvider`）、`IMapFileProvider`（`LocalDiskMapFileProvider`）、`IFactorFileProvider`（`LocalDiskFactorFileProvider`）、`IObjectStore`（`LocalObjectStore`）。

### 命名空间

- `QuantConnect.Lean.Engine.Engine`
- `QuantConnect.Lean.Engine.AlgorithmManager`
- `QuantConnect.Lean.Engine.Setup.BacktestingSetupHandler`
- `QuantConnect.Lean.Engine.Setup.BrokerageSetupHandler`
- `QuantConnect.Lean.Engine.DataFeeds.FileSystemDataFeed`
- `QuantConnect.Lean.Engine.DataFeeds.LiveTradingDataFeed`
- `QuantConnect.Lean.Engine.DataFeeds.Queues.LiveDataQueue`
- `QuantConnect.Lean.Engine.TransactionHandlers.BacktestingTransactionHandler`
- `QuantConnect.Lean.Engine.TransactionHandlers.BrokerageTransactionHandler`
- `QuantConnect.Lean.Engine.Results.BacktestingResultHandler`
- `QuantConnect.Lean.Engine.Results.LiveTradingResultHandler`
- `QuantConnect.Lean.Engine.RealTime.BacktestingRealTimeHandler`
- `QuantConnect.Lean.Engine.RealTime.LiveTradingRealTimeHandler`
- `QuantConnect.Lean.Engine.HistoricalData.SubscriptionDataReaderHistoryProvider`
- `QuantConnect.Lean.Engine.Storage.LocalObjectStore`
- `BrokerageHistoryProvider`（实盘历史，常与本地 reader 组成列表）
- `PaperBrokerage`、`InteractiveBrokersBrokerage`

`live-interactive-iqfeed`：`live-mode-brokerage` = `InteractiveBrokersBrokerage`，`data-queue-handler` = `QuantConnect.IQFeed.IQFeedDataQueueHandler`。下单通道与行情源可以不是同一家。

## 管道

### 时间片

`AlgorithmManager` 反复执行，直到算法结束：

1. `ISynchronizer` 交出一个 `TimeSlice`（多分辨率按算法时钟对齐）。
2. 写入 `Security` 价格，应用分红拆股，推进 Consolidator 与指标。
3. 宇宙数据导致增删证券时，调用 `OnSecuritiesChanged`。
4. 经典算法进入 `OnData(Slice)`。装了 Framework 则走五段模型。
5. `ITransactionHandler.ProcessSynchronousEvents`：撮合或接收券商回报。
6. `IRealTimeHandler` 的定时任务在同一条同步路径上执行。回测对齐算法时间；实盘对齐墙钟。
7. `IResultHandler` 采样权益、日志与图。
8. 未到结束时间则回到步骤 1。结束时 `OnEndOfAlgorithm`，结果落盘或持续外推。

`OnData` 的参数在三种环境下都是 `Slice`。

### 订单

1. 策略调用 `MarketOrder` / `SetHoldings` / `Liquidate`，只生成订单请求。
2. `BuyingPowerModel` 预检保证金。
3. `ITransactionHandler` 接管。
4. 回测与 `live-paper`：`BacktestingTransactionHandler` + `FillModel` 在进程内成交。费用与滑点来自 `IBrokerageModel`。
5. 真仓：`BrokerageTransactionHandler` 把订单交给 `IBrokerage`，以券商回报为成交依据。
6. `OrderEvent` 更新 `Portfolio`，然后 `OnOrderEvent`。

订单三件套：`Order`（请求）、`OrderTicket`（改单撤单句柄）、`OrderEvent`（状态变化）。

### Algorithm Framework

装上模型后，每个切片按此顺序执行。订单仍进入上面的订单管道。

| 顺序 | 接口 | 输出 | 示例 |
| --- | --- | --- | --- |
| 1 | `IUniverseSelectionModel` | 证券集合变化 | `ManualUniverseSelectionModel` |
| 2 | `IAlphaModel` | `Insight`（方向、周期、置信度） | `EmaCrossAlphaModel` |
| 3 | `IPortfolioConstructionModel` | `PortfolioTarget` | `EqualWeightingPortfolioConstructionModel` |
| 4 | `IRiskManagementModel` | 调整后的目标；只削减，不加仓 | `MaximumDrawdownPercentPerSecurity` |
| 5 | `IExecutionModel` | `Order` | `ImmediateExecutionModel` |

经典写法把规则放在 `OnData`，不经过这五段。

## 策略可见类型

| 类型 | 作用 |
| --- | --- |
| `QCAlgorithm` | 基类。`Initialize`、`OnData`、`OnOrderEvent`、`OnSecuritiesChanged`、`OnEndOfDay`、`OnEndOfAlgorithm` |
| `Slice` | 一个时间点的 `TradeBar` / `QuoteBar` / `Tick`，可含分红、拆股、宇宙数据 |
| `Security` | 价格、持仓、交易时段，以及费用、滑点、结算、购买力模型 |
| `Portfolio` | 现金、持仓、权益。成交应用之后才变化 |
| `Universe` | 动态选股。变动走 `OnSecuritiesChanged` |
| `Insight` / `PortfolioTarget` | Framework 的信号与目标权重 |
| 指标 | `SMA` / `EMA` 等。`self.sma` 注册 Consolidator，数据推进后、`OnData` 前更新 |

Python 方法为 snake_case（`set_start_date`、`on_data`），对应 C# PascalCase。引擎本身是 C#。

`IBrokerageModel` 提供 `FeeModel`、`SlippageModel`、`FillModel`、`SettlementModel`、`BuyingPowerModel`。`set_brokerage_model` 只更换这组规则。登录与报单只发生在 `IBrokerage`。

## 不变式

- `live-paper` 的 `live-mode` 为 true，但 `transaction-handler` 仍是 `BacktestingTransactionHandler`，券商对象是 `PaperBrokerage`。
- 真仓环境才使用 `BrokerageTransactionHandler`。
- `IDataFeed` 的实盘实现永远是 `LiveTradingDataFeed`；换的是后面的 `IDataQueueHandler`。
- 回测成交来自 `FillModel`，不是来自 `IBrokerage`。
- 本地日线路径形如 `data/equity/usa/daily/spy.zip`。map / factor 文件处理代码变更与复权。实盘当前 bar 来自 `IDataQueueHandler`。

## 生成策略代码

Python 算法以 `from AlgorithmImports import *` 开头，类继承 `QCAlgorithm`，文件为 `main.py`。方法用 snake_case。C# 文件为 `Main.cs`，同名方法用 PascalCase。引擎只调用这些回调，策略里不要自写时间循环。

交易决策用 `self.time`。它等于当前 Time Frontier，时区默认纽约，可用 `set_time_zone` 修改。不要用 `datetime.now()`。

`initialize` 里写清 `set_start_date`、`set_end_date`、`set_cash`，再 `add_equity` / `add_forex` / `add_crypto` / `add_future` / `add_option` / `add_index` / `add_cfd` / `add_data`。返回值上的 `.symbol` 才是后续主键。`Symbol` 经过 map 文件，代码变更后仍指向同一标的；不要把行情和持仓长期存在裸字符串上。

分辨率只有 `Resolution.TICK`、`SECOND`、`MINUTE`、`HOUR`、`DAILY`。更长周期用 Consolidator 从更短的订阅合成。本地文件必须覆盖所请求的分辨率，路径是 `data/<security-type>/<market>/<resolution>/`，美股日线 ticker 为小写。

### 切片里有什么

`on_data` 每个切片只包含这一刻到达的数据。读取规则：

- `slice.bars`：`TradeBar`。先写 `if symbol in slice.bars`。
- `slice.quote_bars`：买卖报价。
- `slice.ticks`：逐笔。
- `slice.splits`、`slice.dividends`、`slice.delistings`、`slice.symbol_changed_events`：公司行动，不在 `bars` 里。
- `slice.contains_key(symbol)` 只说明该标的有任意数据。它为真时，`slice.bars[symbol]` 仍可能不存在。

某根 K 线没有成交时，默认 fill-forward，用上一根填充。无任何历史成交的标的，`history` 可以是空的。

### 时间语义

算法只能看见 `self.time` 及之前的数据。`history` 和指标都停在这条前沿上。

日内 bar 在本根结束、下一根开始时送达，收盘价此时才知道。日线自 Lean 16570 起默认 `settings.daily_precise_end_time = True`：bar 覆盖真实开收盘，在市场收盘时送达。恢复旧的午夜日线用 `self.settings.daily_precise_end_time = False`。

因此，周五美股日线在周五收盘附近进入 `on_data`。此时再发的市价单，要等市场仍开着或下一交易时段，由 `FillModel` 决定能否成交。只有日线订阅时，不存在盘中价格路径。

宇宙刚加入的证券，同一切片里可能还没有价格。有 `securities[symbol].has_data` 再下单。需要历史价格时用 `set_warm_up`。

### 指标、预热、定时

`self.sma(symbol, period, resolution)` 返回的指标在数据推进之后、`on_data` 之前更新。读 `indicator.is_ready` 和 `indicator.current.value`。

`set_warm_up(bar_count)` 或 `set_warm_up(timedelta)` 会在起始日之前回放数据。预热期间 `is_warming_up` 为真，不能下单。交易逻辑放在预热结束之后，或放进 `on_warmup_finished`。按 bar 根数预热按交易时段取数；按时间长度预热碰到非交易时段会没有 bar。

定时任务：

```python
self.schedule.on(
    self.date_rules.every_day(symbol),
    self.time_rules.before_market_close(symbol, 10),
    self.rebalance)
```

回测里它按算法时间触发，实盘按墙钟触发。

### 下单语义

`set_holdings(symbol, 1)` 的权重是组合价值的比例，`1` 等于 100% 净值，`0.5` 等于 50%。它不是股数。该权重不另乘杠杆。多标的时传入 `PortfolioTarget` 列表，引擎先减仓再加仓。权重换算成数量为 0，或低于 `settings.minimum_order_margin_portfolio_percentage` 时，不下单也不写日志。

`market_order(symbol, quantity)` 的数量是标的单位，股票为股数，正买负卖。`liquidate(symbol)` 平掉该持仓。

数量经过 `BuyingPowerModel`。购买力不足时订单变为 `Invalid`。保证金账户下，默认 `DefaultBrokerageModel`（`BrokerageName.QUANT_CONNECT_BROKERAGE`，默认 `AccountType.MARGIN`）给股票 2 倍、外汇和 CFD 50 倍、其余资产 1 倍。现金账户杠杆为 1。`set_brokerage_model` 同时换上该券商的费用、滑点、成交和结算模型。

成交价由该证券当前的 `FillModel`、有没有 `QuoteBar`、以及市场是否开着共同决定。不要在代码里把“信号价格”写成“成交价格”。

订单状态包括 `Submitted`、`PartiallyFilled`、`Filled`、`Canceled`、`Invalid`。部分成交只在 `on_order_event` 里累加。持仓数量以 `portfolio[symbol].quantity` 为准。

期货和期权的 `add_future` / `add_option` 得到的是规范合约 `Symbol`。可交易的是链上的具体合约。规范符号本身不能当成一个可买卖的价格序列。

### 价格是复权价

美股订阅默认 `DataNormalizationMode.ADJUSTED`，价格已按拆股和分红调整。`RAW` 才是实际成交价，`SPLIT_ADJUSTED` 只调拆股，`TOTAL_RETURN` 把分红再投资进价格。信号、止损和成交假设必须用同一种模式。

代码变更事件在 `slice.symbol_changed_events`。拆股和分红在对应集合里，由 factor / map 文件驱动。

### 回调

| 回调 | 何时 |
| --- | --- |
| `initialize` | 启动一次 |
| `on_data` | 每个有数据的时间片 |
| `on_securities_changed` | 宇宙增删证券 |
| `on_order_event` | 订单状态变化 |
| `on_end_of_day` | 每个证券的当日交易结束 |
| `on_end_of_algorithm` | 回测结束 |
| `on_warmup_finished` | 预热结束 |
| `on_margin_call` | 保证金不足、引擎准备强平 |
| `on_assignment` | 期权被行权 |
| `on_brokerage_disconnect` / `on_brokerage_reconnect` | 实盘连接闪断 |

日志走 `debug`、`log`、`error`、`plot`。这些进入 `IResultHandler`。`quit` 提前结束算法。

`get_parameter("name")` 读取项目 `config.json` 的 `parameters`。`object_store` 在实盘重启后仍保留键值，用来保存状态。`live_mode` 区分回测与实盘。

`history(symbol, periods, resolution)` 只返回前沿之前的数据。Python 里常得到 DataFrame；标的没有数据时结果为空。

## 研究、回测、实盘

`QuantBook` 用于 `lean research` 的笔记本，能取历史和画图，没有 `TransactionHandler`，不能当成会下单的 `QCAlgorithm`。

回测成绩由引擎从 `Portfolio` 采样，夏普、回撤等统计在 `ResultHandler`，不是策略自己算完再写文件。本地结果在 `<项目>/backtests/<时间戳>/`，实盘在 `<项目>/live/<时间戳>/`。

实盘部署时券商持仓会同步进 `Portfolio`。同一份代码在 `live-paper` 仍由 `BacktestingTransactionHandler` 本地模拟成交。真仓才有部分成交、撤单拒绝和断线回调。重启后的内存变量会丢失，需要跨重启的状态放进 `object_store`。

## 工程文件

```text
工作区/
├── lean.json              # 数据目录、组织
├── data/equity/usa/daily/spy.zip
└── SmaCross/
    ├── main.py            # 或 Main.cs
    ├── research.ipynb
    ├── config.json        # algorithm-language、parameters
    ├── backtests/<时间戳>/
    └── live/<时间戳>/
```

项目 `config.json` 的 `algorithm-language` 为 `Python` 或 `CSharp`。引擎环境在 `Launcher/config.json` 的 `environment`，常用名为 `backtesting`、`live-paper`、`live-interactive`。

本地 CLI：`lean project-create`、`lean research`、`lean backtest`、`lean live deploy`、`lean cloud backtest`。本地回测在 Docker 的 LEAN 镜像里跑。QuantConnect 文档要求使用 CLI 的组织处于付费层级。源码可直接运行 `Launcher`。

## 写代码时的判断顺序

1. 是否只订阅了会用到的分辨率，本地数据路径是否存在。
2. 读 bar 前是否判断 `symbol in slice.bars`，指标是否 `is_ready`。
3. 是否避开 `is_warming_up`，以及没有价格的新证券。
4. `set_holdings` 的权重是否按 0 到 1 的组合比例来写。
5. 止损和比较是否与 `DataNormalizationMode` 一致。
6. 日线信号是否接受“收盘后才知道、成交落到后续交易时段”。
7. 期货期权是否交易具体合约，而不是规范 `Symbol`。
8. 实盘需要保留的状态是否进了 `object_store`。
