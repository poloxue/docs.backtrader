---
title: "统计"
weight: 1
---

# 统计

在 backtrader 中运行的策略主要处理数据源和指标。数据源添加到 `Cerebro` 实例中，最终成为策略的输入（解析后作为实例属性提供），而指标由策略本身声明和管理。

到目前为止，所有 backtrader 示例图表都绘制了三样看似理所当然的东西，因为它们在任何地方都没有声明：

- 现金和价值（经纪商中的资金情况）
- 交易（即操作）
- 买/卖订单

它们是观察器，存在于 `backtrader.observers` 子模块中。`Cerebro` 通过参数 `stdstats`（默认：True）自动将它们添加到策略中。

如果使用默认设置，`Cerebro` 会执行以下等效代码：

```python
import backtrader as bt

cerebro = bt.Cerebro()  # 默认参数：stdstats=True

cerebro.addobserver(bt.observers.Broker)
cerebro.addobserver(bt.observers.Trades)
cerebro.addobserver(bt.observers.BuySell)
```

我们先查看带这三个默认观察器的图表（即使没有发出订单，没有交易发生，现金和投资组合价值也没有变化）：

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import backtrader as bt
import backtrader.feeds as btfeeds

if __name__ == '__main__':
    cerebro = bt.Cerebro(stdstats=False)
    cerebro.addstrategy(bt.Strategy)

    data = bt.feeds.BacktraderCSVData(dataname='../../datas/2006-day-001.txt')
    cerebro.adddata(data)

    cerebro.run()
    cerebro.plot()
```

现在将 `stdstats` 设置为 False 来创建 `Cerebro` 实例（也可以在调用 `run` 时完成）：

```python
cerebro = bt.Cerebro(stdstats=False)
```

图表现在有所不同。

## 访问观察器

如上所示，观察器默认已经存在并收集信息，这些信息可用于统计目的。你可以通过策略的 `stats` 属性访问观察器。

这只是一个占位符。回顾上面添加默认观察器的代码：

```python
cerebro.addobserver(backtrader.observers.Broker)
```

显而易见的问题是：如何访问 Broker 观察器。下面是在策略的 `next` 方法中如何做到这一点的示例：

```python
class MyStrategy(bt.Strategy):

    def next(self):
        if self.stats.broker.value[0] < 1000.0:
           print('WHITE FLAG ... I LOST TOO MUCH')
        elif self.stats.broker.value[0] > 10000000.0:
           print('TIME FOR THE VIRGIN ISLANDS ....!!!')
```

`Broker` 观察器与数据、指标和策略本身一样，也是一个线条对象。它包含两条线：

- `cash`
- `value`

## 观察器的实现

实现方式与指标非常相似：

```python
class Broker(Observer):
    alias = ('CashValue',)
    lines = ('cash', 'value')

    plotinfo = dict(plot=True, subplot=True)

    def next(self):
        self.lines.cash[0] = self._owner.broker.getcash()
        self.lines.value[0] = value = self._owner.broker.getvalue()
```

步骤：

1. 从 `Observer`（而不是 `Indicator`）派生
2. 根据需要声明线条和参数（Broker 有两条线但没有参数）
3. 将有一个自动属性 `_owner`，即持有观察器的策略

观察器在以下条件下开始运行：

- 所有指标都已计算完毕
- 策略的 `next` 方法已执行完毕

这意味着在周期结束时，它们观察已发生的情况。在 `Broker` 的情况下，它只是记录每个时间点的经纪商现金和投资组合价值。

## 将观察器添加到策略

如上所述，`Cerebro` 使用 `stdstats` 参数决定是否添加三个默认观察器，从而减轻最终用户的工作负担。

还可以添加其他观察器，无论是与 `stdstats` 一起使用还是替换它们。

让我们来看一个常规策略：当收盘价高于简单移动平均线时买入，低于时卖出。

这里”添加”了一个观察器：

- `DrawDown`，backtrader 生态系统中已有的观察器

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import argparse
import datetime
import os.path
import time
import sys
import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind

class MyStrategy(bt.Strategy):
    params = (('smaperiod', 15),)

    def log(self, txt, dt=None):
        ''' 记录此策略的日志功能 '''
        dt = dt or self.data.datetime[0]
        if isinstance(dt, float):
            dt = bt.num2date(dt)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 主数据上的简单移动平均线
        sma = btind.SMA(period=self.p.smaperiod)

        # 交叉（1：向上，-1：向下）收盘价/移动平均线
        self.buysell = btind.CrossOver(self.data.close, sma, plot=True)

        # 哨兵为 None：允许新订单
        self.order = None

    def next(self):
        # 访问 -1，因为 drawdown[0] 将在“next”之后计算
        self.log('DrawDown: %.2f' % self.stats.drawdown.drawdown[-1])
        self.log('MaxDrawDown: %.2f' % self.stats.drawdown.maxdrawdown[-1])

        # 检查我们是否在市场中
        if self.position:
            if self.buysell < 0:
                self.log('SELL CREATE, %.2f' % self.data.close[0])
                self.sell()
        elif self.buysell > 0:
            self.log('BUY CREATE, %.2f' % self.data.close[0])
            self.buy()

def runstrat():
    cerebro = bt.Cerebro()

    data = bt.feeds.BacktraderCSVData(dataname='../../datas/2006-day-001.txt')
    cerebro.adddata(data)

    cerebro.addobserver(bt.observers.DrawDown)

    cerebro.addstrategy(MyStrategy)
    cerebro.run()

    cerebro.plot()

if __name__ == '__main__':
    runstrat()
```

视觉输出显示了回撤的演变：

部分文本输出：

```
...
2006-12-14T23:59:59+00:00, MaxDrawDown: 2.62
2006-12-15T23:59:59+00:00, DrawDown: 0.22
2006-12-15T23:59:59+00:00, MaxDrawDown: 2.62
2006-12-18T23:59:59+00:00, DrawDown: 0.00
2006-12-18T23:59:59+00:00, MaxDrawDown: 2.62
...
```

注意

如文本输出和代码所示，`DrawDown` 观察器有两条线：

- `drawdown`
- `maxdrawdown`

默认不绘制 `maxdrawdown` 线，但它仍然可供使用。

实际上，`maxdrawdown` 的最后一个值也可以通过一个名称为 `maxdd` 的直接属性（非线条）获取。

## 开发观察器

上面展示了 `Broker` 观察器的实现。要开发有意义的观察器，可以使用以下信息：

- `self._owner` 是当前正在执行的策略

因此策略中的任何内容对观察器都是可用的。

策略中可能有用的默认内部属性：

- `broker`：提供对策略创建订单的经纪商实例的访问。如在 `Broker` 中所见，通过 `getcash` 和 `getvalue` 方法收集现金和投资组合价值。
- `_orderspending`：策略创建的订单列表，这些订单经纪商已通知策略。`BuySell` 观察器遍历此列表，查找已执行的订单（全部或部分），为给定时间点创建平均执行价格（索引 0）。
- `_tradespending`：交易列表（已完成的买入/卖出或卖出/买入对），由订单编译而成。观察器也可以通过 `self._owner.stats` 路径访问其他观察器。

## 自定义 OrderObserver

标准的 `BuySell` 观察器只关心已执行的操作。我们可以创建一个显示订单创建和过期状态的观察器。

为了清晰显示，不会与价格一起绘制，而是在单独的轴上展示。

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import math
import

 backtrader as bt

class OrderObserver(bt.observer.Observer):
    lines = ('created', 'expired',)

    plotinfo = dict(plot=True, subplot=True, plotlinelabels=True)

    plotlines = dict(
        created=dict(marker='*', markersize=8.0, color='lime', fillstyle='full'),
        expired=dict(marker='s', markersize=8.0, color='red', fillstyle='full')
    )

    def next(self):
        for order in self._owner._orderspending:
            if order.data is not self.data:
                continue

            if not order.isbuy():
                continue

            # 只关心“买入”订单，因为策略中的卖出订单是市场订单并会立即执行

            if order.status in [bt.Order.Accepted, bt.Order.Submitted]:
                self.lines.created[0] = order.created.price

            elif order.status in [bt.Order.Expired]:
                self.lines.expired[0] = order.created.price
```

自定义观察器只关心买入订单，因为这是一个只做多的策略。卖出订单是市价单，会立即执行。

## 更新后的策略

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import datetime
import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind
from orderobserver import OrderObserver

class MyStrategy(bt.Strategy):
    params = (
        ('smaperiod', 15),
        ('limitperc', 1.0),
        ('valid', 7),
    )

    def log(self, txt, dt=None):
        ''' 记录此策略的日志功能 '''
        dt = dt or self.data.datetime[0]
        if isinstance(dt, float):
            dt = bt.num2date(dt)
        print('%s, %s' % (dt.isoformat(), txt))

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # 买入/卖出订单已提交/接受至/由经纪人 - 无需执行任何操作
            self.log('ORDER ACCEPTED/SUBMITTED', dt=order.created.dt)
            self.order = order
            return

        if order.status in [order.Expired]:
            self.log('BUY EXPIRED')

        elif order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    'BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))
            else:  # 卖出
                self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))

        # 哨兵为 None：允许新订单
        self.order = None

    def __init__(self):
        # 主数据上的简单移动平均线
        sma = btind.SMA(period=self.p.smaperiod)

        # 交叉（1：向上，-1：向下）收盘价/移动平均线
        self.buysell = btind.CrossOver(self.data.close, sma, plot=True)

        # 哨兵为 None：允许新订单
        self.order = None

    def next(self):
        if self.order:
            # 待处理订单...不做任何事
            return

        # 检查我们是否在市场中
        if self.position:
            if self.buysell < 0:
                self.log('SELL CREATE, %.2f' % self.data.close[0])
                self.sell()
        elif self.buysell > 0:
            plimit = self.data.close[0] * (1.0 - self.p.limitperc / 100.0)
            valid = self.data.datetime.date(0) + datetime.timedelta(days=self.p.valid)
            self.log('BUY CREATE, %.2f' % plimit)
            self.buy(exectype=bt.Order.Limit, price=plimit, valid=valid)

def runstrat():
    cerebro = bt.Cerebro()

    data = bt.feeds.BacktraderCSVData(dataname='../../datas/2006-day-001.txt')
    cerebro.adddata(data)

    cerebro.addobserver(OrderObserver)

    cerebro.addstrategy(MyStrategy)
    cerebro.run()

    cerebro.plot()

if __name__ == '__main__':
    runstrat()
```

结果图表显示多个订单已过期，可以在新子图（红色方块）中看到。此外，还可以看到”创建”和”执行”之间的天数差异。

## 保存/保持统计数据

截至目前，backtrader 尚未实现跟踪观察器值并存储到文件的机制。最好的方法是：

- 在策略的 `start` 方法中打开文件
- 在策略的 `next` 方法中写入值

以 `DrawDown` 观察器为例，可以这样做：

```python
class MyStrategy(bt.Strategy):

    def start(self):
        self.mystats = open('mystats.csv', 'wb')
        self.mystats.write('datetime,drawdown, maxdrawdown\n')

    def next(self):
        self.mystats.write(self.data.datetime.date(0).strftime('%Y-%m-%d'))
        self.mystats.write(',%.2f' % self.stats.drawdown.drawdown[-1])
        self.mystats.write(',%.2f' % self.stats.drawdown.maxdrawdown[-1])
        self.mystats.write('\n')
```

要保存索引 0 的值，可以在所有观察器处理完毕后，将自定义观察器作为系统的最后一个观察器添加，将值写入 CSV 文件。

注意，Writer 功能可以自动执行此任务。
