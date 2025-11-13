---
title: "卖出操作"
weight: 7
---

# 卖出操作

了解如何进入市场（多头，`self.buy`）后，还要一个“退出概念”，以及了解策略是否在市场中。

本节将演示入场后如何退出市场。

退出逻辑很简单，入场持有 5 根 bar 后（在第 6 根 bar 上）退出，无论是盈利还是亏损。另外，为了简化逻辑，仅在未入场时允许买入订单，即如果有持仓或者进行中的订单都不可买入。

## 逻辑逻辑

要完成这个逻辑，要能确认订单成交时间、成交所在位置、当前是否有进行中的订单以及是否有持仓。

### 订单状态

订单确认状态通过 `Strategy` 的订单状态变化 `notify_order` 方法监听。

```python
def notify_order(self, order):
    print(order.status)
```

订单状态有 `Accepted`（已接受）、`Submitted`（已提交）、`Completed`（已成交）、`Margin`（保证金不足）、`Rejected`（已拒绝）。

订单类中除了状态，还包括订单的其他信息，如执行价格 - `order.executed.price`，已执行价值 - `order.executed.value`，订单手续费 - `order.executed.comm`。

更多订单的信息，可查看 [订单- Orders](/backtrader/docs/09-orders/01-general/)。

### 成交位置

策略对象提供了对默认数据源位置的访问，通过 `len(self)` 即可获得。

```python
len(self)
```

next 方法没有提供 "bar索引"，因此很难理解何时经过了 5 根bar，但调用对象的 len 方法，它会告诉你线的长度。

我们只需记下订单完成时的长度，并查看当前长度是否距离其5根bar。

```python
def notify_order(self, order):
    if order.status in [order.Completed]:
      self.bar_executed = len(self)

def next(self):
    # 其他代码
    if len(self) >= (self.bar_executed + 5):
        # 卖出退场
```

此处无 "时间" 或 "时间框架" 含义，仅仅是 bar 的数量。bar可以代表1分钟、1小时、1天、1周或任何其他时间周期。尽管我们知道数据源是每日的，但策略不对其做任何假设。

### 进行中的订单

是否有进行中的订单如何判断？

我们调用买入和卖出方法会返回创建的（尚未执行的）订单。我们可记录创建后的订单，待最终确认后（即非提交中 Submitted 和订单被接受 Accepted）将其清空。而我们只要在每次交易前，检查是否有进行中的订单即可。

```python
def notify_order(self, order):
  if order.status in [order.Submitted, order.Accepted]:
    return

  self.order = None

def next(self):
  # 是否有进行中的订单
  if self.order:
    return

  # 如果满足买入
  self.order = self.buy()
  # 如果满足退出
  self.order = self.sell()
```

### 是否有持仓

是否有持仓的判断比较简单，通过 `self.position` 即可判断。

```python
if not self.position:
  # 入场逻辑
else:
  # 出场逻辑
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
    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return

        if order.status in [order.Completed]:
            if order.isbuy():
                self.log('BUY EXECUTED, %.2f' % order.executed.price)
            elif order.issell():
                self.log('SELL EXECUTED, %.2f' % order.executed.price)

            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order Canceled/Margin/Rejected')

        self.order = None

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
            if len(self) >= (self.bar_executed + 5):
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

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

输出：

```
Starting Portfolio Value: 100000.00
2000-01-03, Close, 27.85
2000-01-04, Close, 25.39
2000-01-05, Close, 24.05
2000-01-05, BUY CREATE, 24.05
2000-01-06, BUY EXECUTED, 23.61
2000-01-06, Close, 22.63
2000-01-07, Close, 24.37
2000-01-10, Close, 27.29
2000-01-11, Close, 26.49
2000-01-12, Close, 24.90
2000-01-13, Close, 24.77
2000-01-13, SELL CREATE, 24.77
2000-01-14, SELL EXECUTED, 25.70
...
...
...
2000-12-15, SELL CREATE, 26.93
2000-12-18, SELL EXECUTED, 28.29
2000-12-18, Close, 30.18
2000-12-19, Close, 28.88
2000-12-20, Close, 26.88
2000-12-20, BUY CREATE, 26.88
2000-12-21, BUY EXECUTED, 26.23
2000-12-21, Close, 27.82
2000-12-22, Close, 30.06
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
Final Portfolio Value: 100018.53
```

现在，系统确实赚到了钱，恭喜我们又进了一步。

