# 开源交易系统说明

# Backtesting & Live Trading

## backtrader

https://github.com/backtrader/backtrader 

**【Star 13.9k】【完成度高】【架构完整】【复杂策略】【强烈推荐】【event-driven】**

自己看代码去，值得学习。因为通用性强，支持特性多，速度就一般般，尤其是当数据特别大的时候。虽然支持非常复杂的策略，但还是需要修改底层源码。同样支持实盘，用实盘做期货和数字货币毫无压力。

## aat

https://github.com/AsyncAlgoTrading/aat

**【Star 652】【很多TODO】【架构基本完整】【可借鉴】【event-driven】**

- `asyncio` 框架
- 将主要的结构体用`C++`实现，通过`pybind11`绑定，主要结构有order, trade, position, order book,  instrument, account。逻辑实现还是在`Python`
- 本地维护一个订单薄【应该是行情系统做的事】
- Manager+事件分发架构。对于任何事件，依次执行`PortfolioManager.onXXX(event)`, `RiskManager.onXXX(event)`, `OrderManager.onXXX(event)`
- PortfolioManager：和投资组合优化无关【应该叫PositionManager，头寸管理器】
- RiskManager：TODO
- Exchange：继承MarketData（市场行情数据）和OrderEntry（下单）两个模块。已经实现
    - 【实盘】coinbase接口
    - 【实盘】IB 接口
    - 【实盘，美股市场】IEX Cloud https://iexcloud.io/
    - 【回测】csv本地数据接口
    - 【模拟】synthetic 合成市场，模拟订单薄
- 整个交易系统靠Exchange.MarketData.tick(trade)驱动，这里的trade表示市场的成交，用来更新本地的订单薄，根据订单薄来下单【高频】【不支持K线驱动】


## QUANTAXIS

https://github.com/yutiansut/QUANTAXIS

**【Star 8.2k】【完成度不高】【简单策略】【event-driven】**

主要思想：将数据流拆分成微服务，用消息列队全部串起来，下单也做成服务。

Python版本代码质量不高，长期未更新。数据用MongoDB、clickhouse保存。订单、持仓、仓位管理都在文件`QifiAccount.py` 。账户运行时数据，需要跟数据库做同步【增加了不一致的可能性】。

只能用于国内A股和期货。

websocket用来远程控制、设置定时任务和查询收益统计。

策略逻辑都在QAStrategy中，实现各种回调函数on_xxx_bar。

消息队列用于实盘，订阅数据、下单。订单列队由第三方服务器[天勤量化 open-trade-gateway](https://github.com/shinnytech/open-trade-gateway)来接受，数据列队由[QUANTAXIS_RealtimeCollector](https://github.com/yutiansut/QUANTAXIS_RealtimeCollector)来分发。【本人觉得完全没必要用消息列队，反而增加了延迟】，期货数据接口来自[ctpbee](https://github.com/ctpbee/ctpbee)，A股没有写。

还有一个大问题，他订单只要发送到消息列队，就默认一定全部成交，但实际可能会失败【导致数据不一致】。

## VNPY

https://github.com/vnpy/vnpy

**【Star 25.6k】【完成度高】【架构完整】【复杂策略】【强烈推荐】【event-driven】**

优点：接口非常丰富。官方文档非常详细，一直在维护中。虽然官方分了很多的模块，但是大致可以分为几类：

- 交易接口类（gateway/api）：丰富
- 数据接口类（database/datafeed）：丰富
- RPC：交易接口、数据接口都可以包装成服务，远程调用（或者再封装成HTTP接口，使用FastAPI Restful，或者封装成websocket），RPC使用的是zmq（ZeroMQ）。
    - 服务器：[vnpy_webtrader](https://github.com/vnpy/vnpy_webtrader)
    - 客户端：[vnpy_rest](https://github.com/vnpy/vnpy_rest)，[vnpy_websocket](https://github.com/vnpy/vnpy_websocket)
- UI界面（chart/studio）
- 策略类（app）：很多示例【包括实盘和回测】
- 管理类（engine）：
    - OmsEngine【核心】：虽然它叫订单管理系统，但是集所有功能于一身，它管理着所有的订单、成交、持仓、账户信息、当前价格、合约、盘口。不管跑多少个策略，策略下的单，都在这个系统里面。
    - [PortfolioEngine](https://github.com/vnpy/vnpy_portfoliomanager) 计算持仓盈亏的
    - RiskEngine：风控。只提供最基本的一些风控。在下单之前，做一系列检查：
        - 单笔委托数量超过最大设定值
        - 今日成交量超过最大设定值
        - 今日撤单量超过最大设定值
        - 每n秒最大下单量
        - 每n秒最大撤单量
        - 在途订单量超过最大设定值
- 回测类（BacktesterEngine）：包括参数优化optimization。重新写了一套管理订单、成交、持仓、盈亏的逻辑，和实盘代码是完全分开的。

缺点：一个模块就是一个project，导致模块之间的有很多重复代码，维护起来就是灾难性的，尤其是策略类的模块，看似拆分很细扩展性强，实则效率低下。比如他每种类型的策略的回测，都要重写BacktestingEngine.py，然而其中大部分代码都是一样的。这种不同策略分不同模块的做法，实际上是一种偷懒行为，新建新类型的策略，只要复制粘贴就行。

应该一个模块一个Python文件，然后全部放在一个project里面，再把重复代码提炼出来。策略和回测逻辑，是可以抽象出来很多公共的底层逻辑的。其实回测和实盘之间都有很多共同的逻辑可以通用，这一点上[backtrader](https://github.com/backtrader/backtrader) 做的可以。

# Live Trading Only

## stock

https://github.com/myhhub/stock

**【A股】【schedule-driven】**

- `cron`定时任务拉取东财、同花顺网站数据，做了一套数据展示的网站。
- 交易引擎（改的 [easyquant](https://github.com/shidenggui/easyquant) ）就是简单的定时任务（开盘、收盘、间隔半分钟/1分钟/5分钟…触发一次），交易接口使用 [easytrader](https://github.com/shidenggui/easytrader)。

## easytrader

https://github.com/shidenggui/easytrader

**【A股】**

- 使用 [pywinauto](https://github.com/pywinauto/pywinauto) 对下单软件GUI进行识别和控制，实现无人监控自动下单。
- 能复制`joinquant`, `ricequant`, `雪球` 的模拟盘操作到实盘。

## easyquant

https://github.com/shidenggui/easyquant

**【A股】【event-driven】【schedule-driven】**

- 交易引擎简单的定时任务+实时行情回调，交易接口使用 [easytrader](https://github.com/shidenggui/easytrader)，行情数接口使用[easyquotation](https://github.com/shidenggui/easyquotation)。

# Backtesting Only

## Backtesting.py

**【Star 5.3k】【值得借鉴】【轻量级】【复杂策略】【schedule-driven】**

https://kernc.github.io/backtesting.py/

- 使用[bokeh](https://bokeh.org/) 绘图
- 主要类：`_Data`、`_Indicator`、`Strategy`、`Order`、`Position`、`Trade`、`_Broker`、`Backtest`。只支持单一的Data。
- 指标在Init的时候，提前算好了
- 支持止盈止损，支持追踪止损
- 架构类似[backtrader](#backtrader)
- 指标优化，使用`scikit-optimize`库

## bt

https://github.com/pmorissette/bt

**【Star 2.1k】【值得借鉴】【复杂策略】【轻量级】【schedule-driven】**

- 特色：algos中所有交易都**模块化**【算法交易】。策略和证券分别用节点`Node`表示，组成树状图，每个策略维持一个algos列表，运行策略即运行该策略上的algos列表，以及递归运行其子节点上的algos列表。algos 模块有：
    - 时间间隔任务、定时任务
    - 选股和权重
    - 仓位再平衡
    - 计算风险，用于展示
- **只支持一个价格close**
- 可实现：买入持有每月调仓、风险平价、固定收入、统计套利、Predicted Tracking Error Rebalance、父子策略、目标波动率、技术指标等
- 非常丰富的测试用例，单元测试和基准测试bench

## vectorbt

https://github.com/polakowo/vectorbt

**【Star 4.1k】【轻量级】【简单策略】【vectorized】**

- 数据提前加载，开平仓信号提前计算好，支持多参数（参数优化 ），然后批量回测。
- 各种数据、数组的处理函数，指标的计算，全部使用numba加速。
- `generic.splitters`模块机器学习数据集划分算法：`RangeSplitter`, `RollingSplitter`, `ExpandingSplitter`。
- `labels`模块：可以给训练集打标签。
- `portfolio`模块：提前计算好仓位，订单、信号都会影响到仓位，都要提前算好
- `records`模块：稀疏数组
- `signals`模块：生成开仓和平仓信号

## zipline-reloaded

https://github.com/stefan-jansen/zipline-reloaded

**【Star 1.2k】【完成度高】【架构完整】【强烈推荐】【schedule-driven】**

代码质量很高，文档和注释非常完整。只支持Daily和Minute。文档说也支持实盘，但是并没有实盘的接口和案例。

核心模块三大类：
- 数据类，底层是用数据库和bcolz，数据格式：按fields分开，比如一个close数据是二维的ndarray，行是时间，列是assets
- Pipeline数据处理流程类，将数据计算分为多个节点（Term）组成单向无环图，支持各种因子、指标、算法等计算
- 交易引擎类，使用bar来驱动运行

## qlib

https://github.com/microsoft/qlib

**【Star 15.1k】【Microsoft出品】【AI】【重量级】【强烈推荐】【schedule-driven】**

AI助力的投研平台，从最底层到上层依次讲解：

- Infrastructure layer 基础设施层，包含：数据服务器，模型训练器，模型存储和管理器。
    - 数据服务器
        - 整个流程支持配置化
        - 原始数据来源各种网站，提供数据HTTP下载的各种网站，数据以二进制文件形式存储在本地，供上层数据分析。qlib对每一列单独存储，本质是调用`numpy.array.tofile()` 接口保存为文件。可以参考`scripts/dump_bin.py`
        - 额外还有一个项目[qlib-server](https://github.com/microsoft/qlib-server)，远程提供数据和缓存。使用 [Flask-SocketIO](https://flask-socketio.readthedocs.io/) 框架。
        - XXXProvider：实现获取数据的接口。主要有两大类，Client和Local，Client是连接远程qlib-server的接口，Local是从本地文件加载的接口
        - DataLoader：从provider中 或者 文件中 获取数据。数据主要有 **日历**，**instruments（证券信息）**，**features（特征，多列字段组合为一个特征）**。**丰富的查询接口，支持条件过滤，支持formulaic表达式（`rule_expression`）**
        - DataHandler：数据预处理，由处理器`processors`组成
        - Dataset：是已经处理好，可供机器学习或者推理的数据
        - 有缓存机制，将中间结果缓存在本地文件，或者 内存。Redis用来提供分布式锁。
    - 模型训练器
        - 训练任务管理器，支持多进程，任务池
        - 每次训练的结果保存为`record`，所有`record`会记录下来，用来分析模型的性能，进行调参。所有record通过`ExperimentManager`进行管理
    - 预置了非常多机器学习模型
- Learning Framework layer 机器学习框架 **PyTorch**
    - 有监督学习和强化学习
- Workflow layer 从市场数据→推理(预测)→构建投资组合(投资决策)→订单执行(回测)
    - 回测**【复杂策略】【schedule-driven】**
        - trade_strategy，做出投资决策`BaseTradeDecision`，买或卖。策略可以包含一个**投资组合优化器【仓位管理】，比如说指数增强策略。**
        - trade_executor，执行决策。支持嵌套执行器和嵌套策略。比如说日内策略，外层是日策略，决定是否开仓，开仓哪些品种，内层是1分钟策略，决定每分钟买入或卖出量（TWAP）。基于预测模型的策略也叫`SignalStrategy`。内嵌策略可以理解为【**算法交易】**。
- Interface layer 分析报告
    - 回测结果
        - `PortfolioMetrics` 总资产、现金、收益率、持仓市值、换手率等。可以绘制出`report_graph`、`risk_analysis_graph`等图表
        - `Indicator` 算法交易过程中记录的一些指标，比如完成率`ffr`，基准价格`base_price`，超出基准价百分比`price advantage`等
    - 风险模型：`RiskModel`、`POETCovEstimator`、`ShrinkCovEstimator`、`StructuredCovEstimator`
    - 预测模型分析报告
