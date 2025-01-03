---
title: "加密货币中的分位仓位"
weight: 3
---

**在 backtrader 中交易加密货币的分数仓位**

首先，让我们用两句话总结一下 backtrader 的工作方式：

它像一个构建工具包，包含一个基本构建模块（Cerebro），可以将许多不同的模块插入其中。

基本分发版包含许多模块，如指标、分析器、观察者、仓位大小计算器、过滤器、数据源、经纪商、佣金/资产信息方案等...

可以轻松地从头开始构建新的模块，或者基于现有模块进行构建。

基本模块（Cerebro）已经实现了一些自动“插拔”，使得用户可以更轻松地使用框架，而不需要关注所有细节。

因此，该框架已预配置以提供默认行为，例如：

- 使用单一的主数据源
- 1天的时间框架/压缩组合
- 10,000 单位的货币
- 股票交易

这些设置可能并不适合每个人，但重要的是：它可以根据每个交易者/程序员的需求进行定制。

**交易股票：整数**

如上所述，默认配置是用于股票交易，当交易股票时，买入/卖出的是完整的股票份额（即：1、2、50、1000 等，而不是像 1.5 或 1001.7589 这样的数量）。

这意味着，当用户在默认配置下执行以下代码时：

```python
def next(self):
    # 将投资组合的 50% 用于购买主资产
    self.order_target_percent(target=0.5)
```

发生的情况是：

系统会计算出需要多少股票份额，以便该资产在投资组合中的价值尽可能接近 50%。

但是，由于默认配置是与股票交易配合使用，结果股票的数量将是一个整数。

**注意**

请注意，默认配置是使用单一的主数据源，因此在调用 `order_target_percent` 时，实际的数据并未指定。当使用多个数据源时，必须指定获取/卖出哪个数据（除非是主数据源）。

**交易加密货币：分数**

显然，在交易加密货币时，即使是小数点后有 20 位数字，也可以购买“半个比特币”。

好消息是，你可以更改与资产相关的信息。这是通过 `CommissionInfo` 家族的可插拔模块实现的。

一些文档：[Docs - Commission Schemes](https://www.backtrader.com/docu/commission-schemes/commission-schemes/)

**注意**

不得不承认，名字不太合适，因为这些方案不仅仅包含佣金信息，还包含其他内容。

在分数方案中，关注的是该方案的 `getsize(price, cash)` 方法，它有如下文档字符串：

```python
返回在给定价格下执行现金操作所需的仓位大小
```

这些方案与经纪商密切相关，并且可以通过经纪商 API 将这些方案添加到系统中。

经纪商文档在这里：[Docs - Broker](https://www.backtrader.com/docu/broker/)

相关方法是：`addcommissioninfo(comminfo, name=None)`。当 `name` 为 None 时，方案会应用到所有资产；如果指定了名称，则方案仅应用于具有特定名称的资产。

**实现分数方案**

这可以通过扩展现有的基础方案（即 `CommissionInfo`）轻松实现。

```python
class CommInfoFractional(bt.CommissionInfo):
    def getsize(self, price, cash):
        '''返回按价格执行现金操作所需的分数大小'''
        return self.p.leverage * (cash / price)
```

就是这样。通过子类化 `CommissionInfo` 并编写一行代码，目标就实现了。由于原始方案定义支持杠杆，因此杠杆也会被考虑进计算中，万一加密货币能够使用杠杆购买（默认值为 1.0，即无杠杆）。

稍后的代码中，方案将按如下方式添加（通过命令行参数控制）：

```python
if args.fractional:  # 如果需要使用分数方案
    cerebro.broker.addcommissioninfo(CommInfoFractional())
```

即：添加一个子类化方案的实例（注意 `()` 用于实例化）。如上所述，`name` 参数未设置，这意味着它将应用于系统中的所有资产。

**测试效果**

下面是一个实现了简单的移动平均交叉策略（用于多头/空头仓位）的完整脚本，可以直接在 shell 中使用。测试的默认数据源来自 backtrader 仓库中的数据源之一。

**整数模式：没有分数 - 没有乐趣**

```bash
$ ./fractional-sizes.py --plot
2005-02-14,3079.93,3083.38,3065.27,3075.76,0.00
2005-02-15,3075.20,3091.64,3071.08,3086.95,0.00
...
2005-03-21,3052.39,3059.18,3037.80,3038.14,0.00
2005-03-21,Enter Short
2005-03-22,Sell Order Completed - Size: -16 @Price: 3040.55 Value: -48648.80 Comm: 0.00
2005-03-22,Trade Opened  - Size -16 @Price 3040.55
2005-03-22,3040.55,3053.18,3021.66,3050.44,0.00
...
```

一个大小为 16 单位的空头交易已经开仓。整个日志（因显而易见的原因未显示）包含了许多其他操作，所有交易都采用整数大小。

**没有分数**

**分数模式运行**

经过子类化和一行代码的修改，分数的目标就实现了...

```bash
$ ./fractional-sizes.py --fractional --plot
2005-02-14,3079.93,3083.38,3065.27,3075.76,0.00
2005-02-15,3075.20,3091.64,3071.08,3086.95,0.00
...
2005-03-21,3052.39,3059.18,3037.80,3038.14,0.00
2005-03-21,Enter Short
2005-03-22,Sell Order Completed - Size: -16.457437774427774 @Price: 3040.55 Value: -50039.66 Comm: 0.00
2005-03-22,Trade Opened  - Size -16.457437774427774 @Price 3040.55
2005-03-22,3040.55,3053.18,3021.66,3050.44,0.00
...
```

胜利了！空头交易已经开仓，并且这次使用的是大小为 -16.457437774427774 的分数仓位。

**分数**

注意，图表中的最终投资组合价值是不同的，因为实际的交易大小不同。

**结论**

是的，backtrader 完全可以做到。通过可插拔/可扩展的构建工具包方式，用户可以轻松地根据交易者/程序员的具体需求定制行为。

**脚本**

```python
#!/usr/bin/env python
# -*- coding: utf-8; py-indent-offset:4 -*-
###############################################################################
# Copyright (C) 2019 Daniel Rodriguez - MIT License
#  - https://opensource.org/licenses/MIT
#  - https://en.wikipedia.org/wiki/MIT_License
###############################################################################
import argparse
import logging
import sys

import backtrader as bt

# This defines not only the commission info, but some other aspects
# of a given data asset like the "getsize" information from below
# params = dict(stocklike=True)  # No margin, no multiplier

class CommInfoFractional(bt.CommissionInfo):
    def getsize(self, price, cash):
        '''Returns fractional size for cash operation @price'''
        return self.p.leverage * (cash / price)


class St(bt.Strategy):
    params = dict(
        p1=10, p2=30,  # periods for crossover
        ma=bt.ind.SMA,  # moving average to use
        target=0.5,  # percentage of value to use
    )

    def __init__(self):
        ma1, ma2 = [self.p.ma(period=p) for p in (self.p.p1, self.p.p2)]
        self.cross = bt.ind.CrossOver(ma1, ma2)

    def next(self):
        self.logdata()
        if self.cross > 0:
            self.loginfo('Enter Long')
            self.order_target_percent(target=self.p.target)
        elif self.cross < 0:
            self.loginfo('Enter Short')
            self.order_target_percent(target=-self.p.target)

    def notify_trade(self, trade):
        if trade.justopened:
            self.loginfo('Trade Opened  - Size {} @Price {}',
                         trade.size, trade.price)
        elif trade.isclosed:
            self.loginfo('Trade Closed

 - Size {} @Price {} Comm: {:.2f}',
                         trade.size, trade.price, trade.commission)

    def logdata(self):
        if self.position:
            self.loginfo('Pos {} Value {:.2f} Cash {:.2f}',
                         self.position.size, self.position.value, self.broker.cash)

    def loginfo(self, *args):
        '''Logging helper'''
        msg = f'{self.datas[0].datetime.datetime(0)} '
        msg += ' '.join([str(arg) for arg in args])
        print(msg)


def run():
    '''Main execution code'''
    parser = argparse.ArgumentParser(
        description="Backtrader with fractional order size"
    )

    parser.add_argument(
        '-p', '--plot', action='store_true', default=False, help="Plot results"
    )
    parser.add_argument(
        '-f', '--fractional', action='store_true', default=False,
        help="Enable fractional order sizes"
    )

    args = parser.parse_args()

    # Create a Cerebro engine
    cerebro = bt.Cerebro()

    # Load data from Yahoo Finance
    data = bt.feeds.YahooFinanceData(dataname='AAPL')

    # Add the data to the engine
    cerebro.adddata(data)

    if args.fractional:  # Fractional orders mode
        cerebro.broker.addcommissioninfo(CommInfoFractional())

    cerebro.addstrategy(St)

    # Set starting cash (USD)
    cerebro.broker.set_cash(100000.0)

    # Set commission (a flat fee per order)
    cerebro.broker.set_commission(commission=0.005)

    # Set slippage model (default to 0.001)
    cerebro.broker.set_slippage_perc(0.001)

    # Set the portfolio value
    cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')

    # Print out the starting portfolio value
    print(f"Starting Portfolio Value: {cerebro.broker.getvalue()}")

    # Run the strategy
    cerebro.run()

    # Print out the final portfolio value
    print(f"Ending Portfolio Value: {cerebro.broker.getvalue()}")

    if args.plot:
        cerebro.plot()


if __name__ == '__main__':
    run()
```
