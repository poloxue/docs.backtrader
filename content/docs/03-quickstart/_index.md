---
title: "快速上手"
weight: 3
---

# 快速上手

**注意**：快速入门指南中使用的数据文件会不定期更新，这意味着调整后的收盘价会变化，其他组件也会随之改变。因此，实际输出可能与文档编写时的内容不同。

## 简要介绍

我们将通过一系列示例（从最简单的示例到完整的策略）来讲解，但在此之前需要了解两个基本概念。

### 线（Lines）

数据源、指标和策略都有 "Line" 线组成。

一条线（Line）是由一系列点组成的，当这些点连在一起时就形成了一条线（Line）。

在谈论市场时，一个数据源通常每天都有以下几个点：

- 开盘价 (Open)
- 最高价 (High)
- 最低价 (Low)
- 收盘价 (Close)
- 成交量 (Volume)
- 未平仓合约 (OpenInterest)

沿时间轴上的 "开盘价" 就是一条线（Line）。因此，一个数据源通常有6条线。如果再考虑“日期时间” (DateTime)（单个点的实际参考），就可以得到 7 条线（Line）。

### 索引0

访问一条线中的值时，当前值使用索引0访问。而 "最后" 一个输出值使用 -1 访问，这与 Python 中的迭代器约定一致（线可以迭代，因此是一种迭代器），在 Python 中，索引 -1 用于访问迭代器或数组中的 "最后" 一项。

在我们的案例中，最后一个输出值就是被访问的值。

因此，索引0紧跟在-1之后，用于访问线中的当前时刻。

考虑到这一点，如果我们在初始化过程中创建了一个简单移动平均线（Simple Moving Average）：

```python
self.sma = SimpleMovingAverage(...)
```

访问当前移动平均值的最简单方法是：

```python
av = self.sma[0]
```

无需知道处理了多少根K线/分钟/天/月，因为“0”唯一标识当前时刻。

按照Python的传统，使用-1访问“最后”一个输出值：

```python
previous_value = self.sma[-1]
```

依此类推，较早的输出值可以使用-2、-3等索引访问。

## 从 0 到 100：示例

### 环境设置

首先配置最基本的策略运行环境，代码如下：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

执行后的输出：

```
Starting Portfolio Value: 10000.00
Final Portfolio Value: 10000.00
```

在此示例中，我们执行如下几个动作：

- import 导入了backtrader，命名为 bt；
- 使用 bt.Cerebro 实例化了 Cerebro 引擎；
- 执行了 Cerebro 实例（循环处理数据）；
- 并打印了 cerebro.broker.getvalue()，即持仓组合的最后价值；

尽管看起来并不复杂，但明确指出了一些重要内容：

- Cerebro 引擎在后台创建了一个broker实例；
- 该实例已经有了一些初始资金；

这种后台broker实例化是平台的常规特性，旨在简化用户操作。如果用户未设置 broker，系统会使用默认 broker。

还有，10,000 货币单位是一些broker的常见起始金额。

### 设置现金

我们可以更改初始金额并重新运行示例，如下代码：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.broker.setcash(100000.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

输出：

```
Starting Portfolio Value: 100000.00
Final Portfolio Value: 100000.00
```

让我们继续。

### 添加数据源

有了钱不是重点，我们的目的是让策略自动操作资产实现资产增值，而资产本质就是数据。

让我们为示例增加资产，即添加数据源：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import datetime  # For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()

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

执行后的输出：

```
Starting Portfolio Value: 100000.00
Final Portfolio Value: 100000.00
```

代码有所增加，更新内容有：

- 基于脚本位置定位示例数据文件路径；
- 使用开始结束时间过滤资产数据；

现在，数据源已创建并添加到 Cerebro 中。但我们的资产并未增值，为什么呢?

**注意**：Yahoo 在线下载的 CSV 数据按日期降序排列，这不是标准约定，故参数 `reverse=True` 是考虑文件中的 CSV 数据是按日期升序排列。

### 第一个策略

我们已经有了钱和要投资标的，即数据源。让我们将添加策略并打印每一天（bar）的 "收盘（Close）" 价格。

DataSeries（数据源中的底层类）对象有别名来访问众所周知的 OHLC（开盘、高、低、收盘）每日值。这将简化打印逻辑的创建。

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

    def next(self):
        self.log('Close, %.2f' % self.dataclose[0])

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
...
...
...
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
Final Portfolio Value: 100000.00
```

让我们解释下这部分实现代码：

- 在`init`方法调用时，策略已经包含一个数据列表（`self.datas`），这是一个标准的Python列表，数据按插入顺序存储。列表中的第一个数据项`self.datas[0]`是默认交易数据，用于同步所有策略元素（它作为系统时钟）。
- 而 `self.dataclose = self.datas[0].close`这行代码保存了对收盘价数据的引用，这样可以更方便地访问收盘价。
- 策略的`next`方法会在每个新的bar上调用，使用系统时钟（即`self.datas[0]`）作为参考。这样，在指标等其他因素需要一些bar的数据来生成输出之前，`next`方法会在每个新的bar上执行。

### 添加策略逻辑

到现在，我们的资金没有任何变化。因为我们没有任何的买卖动作。我将演示一个简单的策略，如果价格连续 3 个交易日下跌即 - 买入！买入！买入！

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

    def next(self):
        self.log('Close, %.2f' % self.dataclose[0])

        if self.dataclose[0] < self.dataclose[-1]:
            if self.dataclose[-1] < self.dataclose[-2]:
                self.log('BUY CREATE, %.2f' % self.dataclose[0])
                self.buy()

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
2000-01-06, Close, 22.63
2000-01-06, BUY CREATE, 22.63
2000-01-07, Close, 24.37
...
...
...
2000-12-20, BUY CREATE, 26.88
2000-12-21, Close, 27.82
2000-12-22, Close, 30.06
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-27, BUY CREATE, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
Final Portfolio Value: 99725.08
```

多个“买入”创建订单被发出，我们的组合价值减少了。显然缺少了一些重要的内容。

订单已创建，但尚未确定其执行时间和价格。

下一个示例将通过监听订单状态通知来完善这个过程。

### 卖出操作

了解如何进入市场（多头）后，需要一个“退出概念”，并了解策略是否在市场中。

幸运的是，策略对象提供了对默认数据源位置的访问。

买入和卖出方法返回创建的（尚未执行的）订单。

订单状态变化将通过`notify`方法通知策略。

退出概念非常简单：

- 在市场中持有5根bar后（在第6根bar上）退出，无论是盈利还是亏损。

请注意，没有“时间”或“时间框架”含义，仅仅是bar的数量。bar可以代表1分钟、1小时、1天、1周或任何其他时间周期。

尽管我们知道数据源是每日的，但策略不对其做任何假设。

为了简化：
- 仅在未入市时允许买入订单

**注意**

`next`方法没有传递“bar索引”，因此似乎很难理解何时经过了5根bar，但这已经在Pythonic方式中建模：调用对象的`len`方法，它会告诉你线的长度。只需记下操作发生时的长度，并查看当前长度是否距离其5根bar。

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

系统确实赚到了钱。

让我们总结一下：

- 在`init`被调用时，策略已经有一个在平台中的数据列表。
- 列表中的第一个数据`self.datas[

0]`是默认交易数据，用于保持所有策略元素同步（它是系统时钟）。
- `self.dataclose = self.datas[0].close`保持对收盘线的引用。以后访问收盘值只需一个间接层。
- 策略的`next`方法将在系统时钟（`self.datas[0]`）的每个bar上调用。这是真的，直到其他因素（如指标）开始产生影响，它们需要一些bar来开始产生输出。

### 添加一些策略逻辑

通过观察一些图表，我们提出了一个疯狂的想法。

如果价格连续3个交易日下跌……买入！买入！买入！

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

执行后的输出：

```
Starting Portfolio Value: 100000.00
2000-01-03, Close, 27.85
2000-01-04, Close, 25.39
2000-01-05, Close, 24.05
2000-01-05, BUY CREATE, 24.05
2000-01-06, BUY EXECUTED, Price: 23.61, Cost: 23.61, Commission 0.02
2000-01-06, Close, 22.63
2000-01-07, Close, 24.37
2000-01-10, Close, 27.29
2000-01-11, Close, 26.49
2000-01-12, Close, 24.90
2000-01-13, Close, 24.77
2000-01-13, SELL CREATE, 24.77
2000-01-14, SELL EXECUTED, Price: 25.70, Cost: 25.70, Commission 0.03
2000-01-14, OPERATION PROFIT, GROSS 2.09, NET 2.04
...
...
...
2000-12-15, SELL CREATE, 26.93
2000-12-18, SELL EXECUTED, Price: 28.29, Cost: 28.29, Commission 0.03
2000-12-18, OPERATION PROFIT, GROSS -0.06, NET -0.12
2000-12-18, Close, 30.18
2000-12-19, Close, 28.88
2000-12-20, Close, 26.88
2000-12-20, BUY CREATE, 26.88
2000-12-21, BUY EXECUTED, Price: 26.23, Cost: 26.23, Commission 0.03
2000-12-21, Close, 27.82
2000-12-22, Close, 30.06
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
2000-12-29, SELL CREATE, 27.41
Final Portfolio Value: 100016.98
```

系统依然赚钱。

让我们总结一下：

- 在`init`被调用时，策略已经有一个在平台中的数据列表。
- 列表中的第一个数据`self.datas[0]`是默认交易数据，用于保持所有策略元素同步（它是系统时钟）。
- `self.dataclose = self.datas[0].close`保持对收盘线的引用。以后访问收盘值只需一个间接层。
- 策略的`next`方法将在系统时钟（`self.datas[0]`）的每个bar上调用。这是真的，直到其他因素（如指标）开始产生影响，它们需要一些bar来开始产生输出。

#### 自定义策略：参数

将一些值硬编码到策略中，并且没有机会轻松更改它们，这显然是不切实际的。参数可以帮助解决这个问题。

定义参数很简单，类似于：

```python
params = (('myparam', 27), ('exitbars', 5),)
```

或者更具吸引力的格式：

```python
params = (
    ('myparam', 27),
    ('exitbars', 5),
)
```

无论使用哪种格式，都允许在将策略添加到Cerebro引擎时对其进行参数化：

```python
# Add a strategy
cerebro.addstrategy(TestStrategy, myparam=20, exitbars=7)
```

**注意**

以下`sizing`方法已被弃用。这部分内容保留在此处，供查看旧版本示例代码的用户参考。源码已更新为使用：

```python
cerebro.addsizer(bt.sizers.FixedSize, stake=10)
```

请阅读关于sizers的章节。

在策略中使用参数很简单，因为它们存储在`params`属性中。例如，如果我们想设置固定持仓量，可以在初始化期间将`stake`参数传递给持仓管理器：

```python
# Set the sizer stake from the params
self.sizer.setsizing(self.params.stake)
```

我们还可以调用`buy`和`sell`方法时传递`stake`参数，将其值设置为`self.params.stake`。

退出逻辑修改如下：

```python
# Already in the market ... we might sell
if len(self) >= (self.bar_executed + self.params.exitbars):
```

考虑到这一切，示例变为：

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

结果已经发生了变化，即使逻辑没有变化。这是真的，但逻辑没有应用于相同数量的bar。

**注意**

正如之前所解释的，平台会在所有指标准备好生成值时首先调用`next`。在这个绘图示例中（图表中非常明显），MACD是最后一个完全准备好的指标（所有3条线都在生成输出）。第一次买入订单不再安排在2000年1月，而是接近2000年2月底。

### 指标

添加指标的灵感来源于PyAlgoTrade的一个示例，使用简单移动平均线（Simple Moving Average）。

- 如果收盘价大于平均值，则买入。
- 如果在市场中，且收盘价小于平均值，则卖出。
- 仅允许一个活跃操作。

大部分现有代码可以保留。让我们在初始化期间添加平均值并保持对其的引用：

```python
self.sma = bt.indicators.MovingAverageSimple(self.datas[0], period=self.params.maperiod)
```

当然，进入和退出市场的逻辑将依赖于平均值。代码中的逻辑如下：

**注意**

起始现金将为1000货币单位，与PyAlgoTrade示例保持一致，不会应用佣金。

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import datetime  # For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt

class TestStrategy(bt.Strategy):
    params = (
        ('maperiod', 15),
    )

    def log(self, txt, dt=None):
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
    cerebro.broker.setcash(1000.0)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    cerebro.broker.setcommission(commission=0.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

执行后的输出：

```
Starting Portfolio Value: 1000.00
2000-01-24, Close, 25.55
2000-01-25, Close, 26.61
2000-01-25, BUY CREATE, 26.61
2000-01-26, BUY EXECUTED, Size 10, Price: 26.76, Cost: 267.60, Commission 0.00
2000-01-26, Close, 25.96
2000-01-27, Close, 24.43
2000-01-27, SELL CREATE, 24.43
2000-01-28, SELL EXECUTED, Size 10, Price: 24.28, Cost: 242.80, Commission 0.00
2000-01-28, OPERATION PROFIT, GROSS -24.80, NET -24.80
...
...
...
2000-12-20, SELL CREATE, 26.88
2000-12-21, SELL EXECUTED, Size 10, Price: 26.23, Cost: 262.30, Commission 0.00
2000-12-21, OPERATION PROFIT, GROSS -20.60, NET -20.60
2000-12-21, Close, 27.82
2000-12-21, BUY CREATE, 27.82
2000-12-22, BUY EXECUTED, Size 10, Price: 28.65, Cost: 286.50, Commission 0.00
2000-12-22, Close, 30.06
2000-12-26, Close,

 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
2000-12-29, SELL CREATE, 27.41
Final Portfolio Value: 973.90
```

系统变成亏损了。

**注意**

与PyAlgoTrade相同逻辑和数据略有不同结果（略有偏差）。查看完整打印输出可发现一些操作并不完全相同，原因通常是舍入误差。

PyAlgoTrade在将除以调整后收盘价应用于数据时不会舍入数据值。

backtrader提供的Yahoo数据源将值舍入到2位小数。在打印值时看起来相同，但显然有时第五位小数起作用。

舍入到2位小数更为现实，因为市场交易所对每种资产允许的小数位数有限（股票通常为2位小数）。

**注意**

从版本1.8.11.99开始，Yahoo数据源允许指定是否进行舍入以及舍入的小数位数。

### 可视化检查：绘图

打印或记录系统在每个bar时刻的实际情况是很好的，但人类倾向于视觉，因此提供图表视图是正确的。

**注意**

要绘图，需要安装matplotlib。

再次强调，绘图的默认设置可以帮助用户。绘图是一行代码操作：

```python
cerebro.plot()
```

确保在调用`cerebro.run()`之后执行。

为了展示自动绘图功能和一些简单的自定义，我们将执行以下操作：

- 添加第二个移动平均线（指数）。默认情况下，它会与数据一起绘制。
- 添加第三个移动平均线（加权）。自定义为在自己的子图中绘制（即使没有意义）。
- 添加一个慢速随机指标。不更改默认设置。
- 添加一个MACD。不更改默认设置。
- 添加一个RSI。不更改默认设置。
- 在RSI上应用一个简单移动平均线。不更改默认设置（与RSI一起绘制）。
- 添加一个平均真实范围（ATR）。更改默认设置以避免绘图。

策略的`__init__`方法中添加的所有内容：

```python
# Indicators for the plotting show
bt.indicators.ExponentialMovingAverage(self.datas[0], period=25)
bt.indicators.WeightedMovingAverage(self.datas[0], period=25).subplot = True
bt.indicators.StochasticSlow(self.datas[0])
bt.indicators.MACDHisto(self.datas[0])
rsi = bt.indicators.RSI(self.datas[0])
bt.indicators.SmoothedMovingAverage(rsi, period=10)
bt.indicators.ATR(self.datas[0]).plot = False
```

**注意**

即使没有将指标显式添加到策略的成员变量中（例如`self.sma = MovingAverageSimple…`），它们也会自动注册到策略中，并影响`next`的最小周期，并成为绘图的一部分。

在示例中，只有RSI被添加到临时变量rsi中，其唯一目的是在其上创建一个平滑移动平均线（SmoothedMovingAverage）。

示例如下：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import datetime  # For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt

class TestStrategy(bt.Strategy):
    params = (
        ('maperiod', 15),
    )

    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.buyprice = None
        self.buycomm = None
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod)

        # Indicators for the plotting show
        bt.indicators.ExponentialMovingAverage(self.datas[0], period=25)
        bt.indicators.WeightedMovingAverage(self.datas[0], period=25,
                                            subplot=True)
        bt.indicators.StochasticSlow(self.datas[0])
        bt.indicators.MACDHisto(self.datas[0])
        rsi = bt.indicators.RSI(self.datas[0])
        bt.indicators.SmoothedMovingAverage(rsi, period=10)
        bt.indicators.ATR(self.datas[0], plot=False)

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
    cerebro.broker.setcash(1000.0)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    cerebro.broker.setcommission(commission=0.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())

    # Plot the result
    cerebro.plot()
```

执行后的输出：

```
Starting Portfolio Value: 1000.00
2000-02-18, Close, 27.61
2000-02-22, Close, 27.97
2000-02-22, BUY CREATE, 27.97
2000-02-23, BUY EXECUTED, Size 10, Price: 28.38, Cost: 283.80, Commission 0.00
2000-02-23, Close, 29.73
...
...
...
2000-12-21, BUY CREATE, 27.82
2000-12-22, BUY EXECUTED, Size 10, Price: 28.65, Cost: 286.50, Commission 0.00
2000-12-22, Close, 30.06
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
2000-12-29, SELL CREATE, 27.41
Final Portfolio Value: 981.00
```

### 优化

许多交易书籍中提到，每个市场和每个交易的股票（或商品等）都有不同的节奏。没有一种适合所有的策略。

在绘图示例之前，当策略开始使用一个指标时，其周期默认值为15个bar。这是一个策略参数，可以用于优化，改变参数值以找出哪个更适合市场。

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

### 结论

逐步示例展示了如何从一个简单的脚本逐步发展为一个完整的交易系统，该系统甚至可以绘制结果并进行优化。

可以做更多的事情来提高获胜机会：

- 自定义指标
- 创建指标很容易（甚至绘图也很容易）
- 持仓管理器
- 资金管理对许多人来说是成功的关键
- 订单类型（限价单、止损单、止损限价单）
- 其他

为了确保所有上述项目都能充分利用，文档提供了对它们（以及其他主题）的深入了解。

查看目录并继续阅读……并进行开发。

祝你好运！

