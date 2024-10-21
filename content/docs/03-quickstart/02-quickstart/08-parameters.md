---
title: "参数定义"
weight: 8
---

# 参数定义

前面的案例中，我们都是将参数硬编码到策略中，要修改参数就要修改策略代码。本小节，我们将学习如何在 backtrader 自定义参数。

## 定义参数

策略参数的定义非常简单，如在策略中定义两个参数：`myparam` 和 `exitbars`。

```python
class TestStrategy:
    params = (('myparam', 27), ('exitbars', 5),)
```

如上，参数 `myparam` 的默认值是 27，而 `exitbars` 的默认值是 5。

## 配置参数

参数在 Cerebro 添加策略时配置。

```python
# Add a strategy
cerebro.addstrategy(TestStrategy, myparam=20, exitbars=7)
```

## 使用参数

在策略中，我们直接通过 `self.params.param_name ` 的形式即可调用参数。如用参数 `exitbars` 修改退出逻辑：

```python
# Already in the market ... we might sell
if len(self) >= (self.bar_executed + self.params.exitbars):
```

## 完整示例

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import datetime  # For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt

class TestStrategy(bt.Strategy):
    params = (
        ('exitbars', 5),
    )

    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.buyprice = None
        self.buycomm = None

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
        self.log('OPERATION PROFIT, GROSS

 %.2f, NET %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        self.log('Close, %.2f' % self.dataclose[0])

        if self.order:
            return

        if not self.position:
            if self.dataclose[0] < self.dataclose[-1]:
                if self.dataclose[-1] < self.dataclose[-2]:
                    self.log('BUY CREATE, %.2f' % self.dataclose[0])
                    self.order = self.buy()
        else:
            if len(self) >= (self.bar_executed + self.params.exitbars):
                self.log('SELL CREATE, %.2f' % self.dataclose[0])
                self.order = self.sell()

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.addstrategy(TestStrategy)

    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        fromdate=datetime.datetime(2000, 1, 1),
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    cerebro.adddata(data)
    cerebro.broker.setcash(100000.0)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

执行后的输出：

```
Starting Portfolio Value: 100000.00
2000-01-03, Close, 27.85
2000-01-04, Close, 25.39
2000-01-05, Close, 24.05
2000-01-05, BUY CREATE, 24.05
2000-01-06, BUY EXECUTED, Size 10, Price: 23.61, Cost: 236.10, Commission 0.24
2000-01-06, Close, 22.63
...
...
...
2000-12-20, BUY CREATE, 26.88
2000-12-21, BUY EXECUTED, Size 10, Price: 26.23, Cost: 262.30, Commission 0.26
2000-12-21, Close, 27.82
2000-12-22, Close, 30.06
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
2000-12-29, SELL CREATE, 27.41
Final Portfolio Value: 100169.80
```

结果已经发生了变化，即使逻辑没有变化。

