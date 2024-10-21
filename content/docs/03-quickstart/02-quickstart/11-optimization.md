---
title: "策略优化"
weight: 11
---

# 优化

许多交易书籍中提到，每个市场和每个交易的股票（或商品等）都有不同的节奏，没有一种适合所有的策略。

在绘图示例前，当策略开始使用一个指标，周期默认值为 15 个 bar。这是一个策略参数，可以用于优化，改变参数值以找出哪个更适合你的市场。

**注意**

关于优化及其优缺点的文献很多。但建议总是指向同一个方向：不要过度优化。如果交易思路不健全，优化可能会产生一个仅对回测数据集有效的正面结果。

示例修改为优化简单移动平均线的周期。为了清晰起见，已删除与买卖订单相关的任何输出。

示例如下：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import datetime  #

 For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt

class TestStrategy(bt.Strategy):
    params = (
        ('maperiod', 15),
        ('printlog', False),
    )

    def log(self, txt, dt=None, doprint=False):
        if self.params.printlog or doprint:
            dt = dt or self.datas[0].datetime.date(0)
            print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.buyprice = None
        self.buycomm = None
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod)

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return

        if order.status in [order.Completed]:
            if order.isbuy():
                self.log('BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:
                self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order Canceled/Margin/Rejected')

        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('OPERATION PROFIT, GROSS %.2f, NET %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        self.log('Close, %.2f' % self.dataclose[0])

        if self.order:
            return

        if not self.position:
            if self.dataclose[0] > self.sma[0]:
                self.log('BUY CREATE, %.2f' % self.dataclose[0])
                self.order = self.buy()
        else:
            if self.dataclose[0] < self.sma[0]:
                self.log('SELL CREATE, %.2f' % self.dataclose[0])
                self.order = self.sell()

    def stop(self):
        self.log('(MA Period %2d) Ending Value %.2f' %
                 (self.params.maperiod, self.broker.getvalue()), doprint=True)

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    strats = cerebro.optstrategy(
        TestStrategy,
        maperiod=range(10, 31))

    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        fromdate=datetime.datetime(2000, 1, 1),
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    cerebro.adddata(data)
    cerebro.broker.setcash(1000.0)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    cerebro.broker.setcommission(commission=0.0)

    cerebro.run(maxcpus=1)
```

不再调用`addstrategy`添加策略类，而是调用`optstrategy`。并且传递一个值范围，而不是单一值。

添加了一个策略钩子`stop`方法，当数据耗尽且回测结束时将调用它。用于在broker中打印组合的最终净值（之前在Cerebro中完成）。

系统将为每个范围值执行策略。输出如下：

```
2000-12-29, (MA Period 10) Ending Value 880.30
2000-12-29, (MA Period 11) Ending Value 880.00
2000-12-29, (MA Period 12) Ending Value 830.30
2000-12-29, (MA Period 13) Ending Value 893.90
2000-12-29, (MA Period 14) Ending Value 896.90
2000-12-29, (MA Period 15) Ending Value 973.90
2000-12-29, (MA Period 16) Ending Value 959.40
2000-12-29, (MA Period 17) Ending Value 949.80
2000-12-29, (MA Period 18) Ending Value 1011.90
2000-12-29, (MA Period 19) Ending Value 1041.90
2000-12-29, (MA Period 20) Ending Value 1078.00
2000-12-29, (MA Period 21) Ending Value 1058.80
2000-12-29, (MA Period 22) Ending Value 1061.50
2000-12-29, (MA Period 23) Ending Value 1023.00
2000-12-29, (MA Period 24) Ending Value 1020.10
2000-12-29, (MA Period 25) Ending Value 1013.30
2000-12-29, (MA Period 26) Ending Value 998.30
2000-12-29, (MA Period 27) Ending Value 982.20
2000-12-29, (MA Period 28) Ending Value 975.70
2000-12-29, (MA Period 29) Ending Value 983.30
2000-12-29, (MA Period 30) Ending Value 979.80
```

### 结果

对于小于18的周期，策略（无佣金）亏损。
对于18到26（包括）的周期，策略盈利。
超过26的周期，策略再次亏损。
对于此策略和给定数据集，获胜的周期是：

20个bar，盈利78.00单位货币（即7.8%）。

**注意**

绘图示例中的额外指标已被移除，操作开始仅受正在优化的简单移动平均线影响。因此，周期15的结果略有不同。
