---
title: "介绍"
weight: 1
---

## 什么是 Backtrader？

Backtrader 是一个基于 Python 实现功能强大且开源的量化回测交易框架。Backtrader 有着完整的基础设施，支持编写可重用组件，如策略、指标和分析器。

Backtrader 支持多品种多周期多策略的回测。相对于其他专门用于策略回测的框架，Backtrader 不仅仅是可以回测交易策略，还可以连接 Broker 实盘交易。

## 有哪些特性呢？

稍微展开说说 Backtrader 的特性吧。

![](https://cdn.jsdelivr.net/gh/poloxue/images@backtrader/00-overview-01.png)

**丰富的技术指标**

Backtrader 提供超过 122 种内置指标，包括多种移动平均线（SMA、EMA 等）和经典指标（MACD、随机指标、RSI 等）。

如果内置指标不能满足需求，Backtrader 还集成了其他技术指标库，如 ta-lib 库。而且，我们也可以自定义技术指标，Backtrader 也提供了自定义指标的能力。

**支持多种数据源**

Backtrader 支持多种数据源类型，如常见的一些数据源：

- CSV 数据文件；
- Pandas 数据源，可以借助 Pandas 的强大功能连接各种不同的数据源；
- YahooFinance，这个好像已经不能用了，不过借助 yfinance 包和 pandas 可轻易集成；
- 其他的一些公开数据源。

除此以外，Backtrader 支持自定义数据格式，如希望实现股票选择策略，可自定义数据格式，加入财务数据和因子，如 PE、PB、ROI 等指标。

Backtrader 支持同时添加多个 DataFeed 数据源，如多个时间框架数据源实现多周期策略，多标的数据源实现组合交易策略，或是直接进行数据的重采样和重放功能，实现真实交易环境的模拟。

**灵活的策略实现**

Backtrader 支持灵活的策略开发，如在策略初始化阶段，Backtrader 支持预热计算，预热完成才进入交易逻辑部分。

Backtrader 支持并行运行多个策略，实现多策略组合。其内置的 Broker 支持多种订单类型，如市价单、限价单、止损单等，支持多空交易，自定义佣金方案和信用利息，提供针对期货类工具的连续现金调整，支持基金模式和自定义滑点策略。

它还提供多种订单生成方法，并支持事件通知机制，包括新数据、订单、交易和计时器等等。

**性能分析与绘图**

Backtrader 内置多个性能分析器，包括时间收益、交易分析器、夏普比率、VWR 和 SQN 等，帮助用户全面评估策略表现。Backtrader 支持通过一行代码绘图（安装 Matplotlib），用户可自定义绘图，评估策略表现。

## 架构设计

Backtrader 本身是一个模块拆分的非常优秀的框架，解耦复用方面做的非常优秀。

如下所示的一系列组件：

* **DataFeed**：负责加载行情数据，包括历史数据和实时数据，是策略运行的基础输入。
* **Cerebro**：Backtrader 的核心引擎，用于统一管理数据、策略、分析器、经纪商等组件，并负责执行回测或实盘。
* **Indicator**：用于计算常见的技术指标（如均线、RSI、布林带等），也可以自定义指标逻辑，为策略提供信号依据。
* **Strategy**：定义具体的交易规则和执行逻辑，包括开仓、平仓、止损止盈等，是系统的决策核心。
* **Broker**：模拟或连接真实交易账户，负责订单撮合、资金管理和手续费处理。
* **Analyzer**：对策略的回测结果进行统计与评估，如收益率、最大回撤、夏普比率等。
* **Observer**：用于跟踪策略运行过程并输出可视化结果，如资金曲线、买卖点、回撤变化等。

如下是各个组件之间的关系架构图：

![](https://cdn.jsdelivr.net/gh/poloxue/images@backtrader/00-overview-02.png)

Backtrader 的核心 **Cerebro** 调度整个回测流程。

首先，由 **DataFeed** 提供行情数据，Cerebro 将数据传递给 **Strategy**。策略读取数据并调用 **Indicator** 计算技术信号，据此生成买卖决策并交由 **Broker** 执行交易。交易完成后，Broker 更新账户与持仓，并将结果反馈给 Cerebro。

随后，**Analyzer** 分析策略绩效（如收益、回撤），**Observer** 记录并可视化资金曲线与交易点。Cerebro 不断循环此流程，直到数据结束，最终汇总输出策略表现。

整个系统形成从“数据→决策→执行→反馈→分析”的闭环结构。

## 实盘交易

Backtrader 的交易逻辑和经纪商 Broker 的操作是逐事件驱动，这让其可非常容易实现实盘交易。

我们在回测时，指标计算尽可能通过向量化计算，预加载源数据提升计算速度。而实盘模式下，可在仅事件模式下运行，无需预加载数据。

Backtrader 支持与多种 Broker 经纪商实时交易，包括 Interactive Brokers、Oanda v1 和 VisualChart。此外，还支持第三方经纪商如 Alpaca、Oanda v2 和 ccxt 的集成。其中，ccxt 是加密货币实盘交易的 Python 库。

加密货币的程序交易账号相对更加容易获取，后续或许我会用 ccxt 演示 backtrader 的实盘交易加密货币。
