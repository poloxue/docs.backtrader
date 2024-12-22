---
title: "参数定义"
weight: 8
---

# 参数定义

前面的案例中，参数都是硬编码在策略中。本节将介绍如何在 backtrader 自定义参数。

## 定义参数

策略参数的定义非常简单，如在策略中定义两个参数：`myparam` 和 `exitbars`。

```python
class TestStrategy:
    params = (('myparam', 27), ('exitbars', 5),)
```

参数 `myparam` 的默认值是 27，`exitbars` 的默认值是 5。

## 配置参数

我们可以在添加策略时修改参数默认值。

```python
# Add a strategy
cerebro.addstrategy(TestStrategy, myparam=20, exitbars=7)
```

## 使用参数

策略代码中直接通过 `self.params.param_name ` 即可调用参数。

如下代码，通过参数 `exitbars` 修改退出逻辑：

```python
if len(self) >= (self.bar_executed + self.params.exitbars):
```

## 完整示例

```python
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
```
