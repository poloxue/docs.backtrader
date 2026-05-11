---
title: "Analyzers"
weight: 1
---

# 分析器

无论是回测还是交易，分析交易系统的表现至关重要：不仅要看是否获利，还要看实现利润的过程中是否承担了过多风险，或者与参考资产（或无风险资产）相比是否真的值得。

这就是分析器的作用：提供对已发生或当前情况的分析。

分析器的设计参照了线条对象（例如具有 `next` 方法），但有一个主要区别：分析器不包含线条。

这意味着它们不太消耗内存，即使在分析了成千上万个价格柱之后，可能仍然只在内存中保存单个结果。

## 在生态系统中的位置

分析器（如同策略、观察者和数据）通过 `cerebro` 实例添加到系统中：

```python
addanalyzer(ancls, *args, **kwargs)
```

但在 `cerebro.run` 期间，对于系统中的每个策略，会发生以下情况：

- `ancls` 会用 `*args` 和 `**kwargs` 实例化
- 实例会附加到策略上

这意味着：

如果回测运行包含 3 个策略，将创建 3 个 `ancls` 实例，每个实例附加到不同的策略上。

结论：分析器分析的是单个策略的表现，而非整个系统。

## 附加位置

某些分析器可能会使用其他分析器来完成其工作。例如：`SharpeRatio` 使用 `TimeReturn` 的输出进行计算。

这些子分析器也会插入到创建它们的策略中，但对用户是完全不可见的。

## 属性

分析器提供了一些自动设置的默认属性，方便使用：

- `self.strategy`：对当前运行策略的引用。策略能访问的任何内容，分析器也能访问。
- `self.datas[x]`：策略中的数据源数组。虽然可以通过策略引用访问，但快捷方式更方便。
- `self.data`：`self.datas[0]` 的快捷方式。
- `self.dataX`：对应 `self.datas[x]` 的快捷方式。

还有一些其他别名可用：

- `self.dataX_Y`，其中 X 对应 `self.datas[X]`，Y 对应线条，最终指向 `self.datas[X].lines[Y]`。
如果线条有名称，还可以使用以下方式：

- `self.dataX_Name`，解析为 `self.datas[X].Name`，按名称而非索引返回线条。
对于第一个数据，最后两个快捷方式可以省略 X 数字。例如：

- `self.data_2` 指 `self.datas[0].lines[2]`。
- `self.data_close` 指 `self.datas[0].close`。

## 返回分析结果

`Analyzer` 基类创建了 `self.rets`（类型为 `collections.OrderedDict`）成员属性来返回分析结果。这是在 `create_analysis` 方法中完成的，子类可以覆盖该方法。

## 操作模式

虽然分析器不是线条对象，不迭代线条，但它们遵循相同的操作模式。

- 在系统启动前实例化（调用 `__init__`）。
- 使用 `start` 信号表示操作开始。
- `prenext` / `nextstart` / `next` 方法会根据策略的计算最小周期被调用。
- `prenext` 和 `nextstart` 的默认行为是调用 `next`，因为分析器可能从系统启动的第一刻就开始分析。
- 在线条对象中通常调用 `len(self)` 检查实际条数。这在分析器中同样适用，返回 `self.strategy` 的值。
- 订单和交易通过 `notify_order` 和 `notify_trade` 进行通知，与策略相同。
- 现金和价值通过 `notify_cashvalue` 方法通知，与策略相同。
- 现金、价值、基金价值和基金份额通过 `notify_fund` 方法通知，与策略相同。
- 使用 `stop` 信号表示操作结束。
- 常规操作周期完成后，分析器提供额外的提取/输出方法：
  - `get_analysis`：通常（非强制）返回包含分析结果的字典样对象。
  - `print`：使用标准的 `backtrader.WriterFile`（除非被覆盖）输出 `get_analysis` 的结果。
  - `pprint`：使用 Python 的 `pprint` 模块打印 `get_analysis` 结果。

最后：

`get_analysis` 创建一个类型为 `collections.OrderedDict` 的成员属性 `self.ret`，分析器将结果写入其中。

子类可以覆盖此方法以改变行为。

## 分析器模式

在 backtrader 中开发分析器揭示了两种不同的使用模式：

- 通过 `notify_xxx` 和 `next` 方法在执行过程中收集信息，并在 `next` 中生成当前分析。
  - 例如，`TradeAnalyzer` 仅使用 `notify_trade` 方法生成统计数据。
- 按上述方式收集信息（或不收集），但在 `stop` 方法中进行一次性分析。
  - `SQN`（系统质量编号）在 `notify_trade` 中收集交易信息，但在 `stop` 方法中生成统计数据。

## 快速示例

尽可能简单：

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import datetime
import backtrader as bt
import backtrader.analyzers as btanalyzers
import backtrader.feeds as btfeeds
import backtrader.strategies as btstrats

cerebro = bt.Cerebro()

# 数据
dataname = '../datas/sample/2005-2006-day-001.txt'
data = btfeeds.BacktraderCSVData(dataname=dataname)

cerebro.adddata(data)

# 策略
cerebro.addstrategy(btstrats.SMA_CrossOver)

# 分析器
cerebro.addanalyzer(btanalyzers.SharpeRatio, _name='mysharpe')

thestrats = cerebro.run()
thestrat = thestrats[0]

print('Sharpe Ratio:', thestrat.analyzers.mysharpe.get_analysis())
```

执行它（已将其存储在 analyzer-test.py 中）：

```sh
$ ./analyzer-test.py
Sharpe Ratio: {'sharperatio': 11.647332609673256}
```

没有绘图，因为 `SharpeRatio` 是计算结束时的单一值。

## 分析器解析

重申一下，分析器不是线条对象，但为了无缝集成到 backtrader 生态系统中，遵循了线条对象的内部 API 约定（实际上是混合模式）。

注意

`SharpeRatio` 的代码已经演变，例如考虑了年化，此版本仅供参考。

请查看分析器参考。

此外，`SharpeRatio_A` 直接以年化形式返回值，而不受时间范围影响。

`SharpeRatio` 的基础代码（简化版本）：

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import operator
from backtrader.utils.py3 import map
from backtrader import Analyzer, TimeFrame
from backtrader.mathsupport import average, standarddev
from backtrader.analyzers import AnnualReturn

class SharpeRatio(Analyzer):
    params = (('timeframe', TimeFrame.Years), ('riskfreerate', 0.01),)

    def __init__(self):
        super(SharpeRatio, self).__init__()
        self.anret = AnnualReturn()

    def start(self):
        # Not needed ... but could be used
        pass

    def next(self):
        # Not needed ... but could be used
        pass

    def stop(self):
        retfree = [self.p.riskfreerate] * len(self.anret.rets)
        retavg = average(list(map(operator.sub, self.anret.rets, retfree)))
        retdev = standarddev(self.anret.rets)
        self.ratio = retavg / retdev

    def get_analysis(self):
        return dict(sharperatio=self.ratio)
```

代码分解说明：

- `params` 声明
  - 虽然声明的参数没有使用（作为示例），但分析器像大多数其他对象一样支持参数。
- `__init__` 方法
  - 就像策略在 `__init__` 中声明指标一样，分析器声明辅助对象。
  - 这里：`SharpeRatio` 使用年度回报计算。计算会自动进行，结果可供 `SharpeRatio` 使用。
- `next` 方法
  - `SharpeRatio` 不需要它，但此方法会在父策略每次调用 `next` 后被调用。
- `start` 方法
  - 在回测开始前调用，可用于额外的初始化任务。

`SharpeRatio` 不需要它。
- `stop` 方法
  - 在回测结束后调用。如 `SharpeRatio` 所做的那样，可用于完成计算。
- `get_analysis` 方法（返回字典）
  - 为外部调用者提供分析结果。

返回：包含分析结果的字典。

## 参考

```python
class backtrader.Analyzer()
```

`Analyzer` 基类。所有分析器都是此类的子类。

分析器实例在策略的框架内运行，为该策略提供分析。

自动设置的成员属性：

- `self.strategy`（提供对策略及其可访问内容的访问）
- `self.datas[x]` 提供对系统中数据源数组的访问，也可通过策略引用访问
- `self.data` 提供对 `self.datas[0]` 的访问
- `self.dataX` -> `self.datas[X]`
- `self.dataX_Y` -> `self.datas[X].lines[Y]`
- `self.dataX_name` -> `self.datas[X].name`
- `self.data_name` -> `self.datas[0].name`
- `self.data_Y` -> `self.datas[0].lines[Y]`

这不是线条对象，但方法和操作遵循相同的设计：

- `__init__`：实例化和初始设置
- `start` / `stop`：表示操作的开始和结束
- `prenext` / `nextstart` / `next`：方法家族，遵循策略中相同方法的调用
- `notify_trade` / `notify_order` / `notify_cashvalue` / `notify_fund`：接收与策略等效方法相同的通知

操作模式是开放的，没有首选模式。分析可以通过 `next` 调用生成，在 `stop` 期间生成，或通过单个方法如 `notify_trade` 生成。

关键是覆盖 `get_analysis` 以返回包含分析结果的字典样对象（实际格式取决于实现）。

- `start()`
  - 调用以指示操作开始，给分析器时间设置所需内容。
- `stop()`
  - 调用以指示操作结束，给分析器时间关闭所需内容。
- `prenext()`
  - 在策略的每次 `prenext` 调用期间调用，直到达到策略的最小周期。
  - 分析器的默认行为是调用 `next`。
- `nextstart()`
  - 在策略的 `nextstart` 调用时调用一次，当首次达到最小周期时。
- `next()`
  - 在策略的每次 `next` 调用期间调用，一旦达到策略的最小周期。
- `notify_cashvalue(cash, value)`
  - 在每个 `next` 周期之前接收现金/价值通知。
- `notify_fund(cash, value, fundvalue, shares)`
  - 接收当前现金、价值、基金价值和基金份额。
- `notify_order(order)`
  - 在每个 `next` 周期之前接收订单通知。
- `notify_trade(trade)`
  - 在每个 `next` 周期之前接收交易通知。
- `get_analysis()`
  - 返回包含分析结果的字典样对象。
  - 字典中分析结果的键和值的格式取决于实现。
  - 甚至不强制结果是字典样对象，只是一种约定。
  - 默认实现返回由默认 `create_analysis` 方法创建的默认 `OrderedDict rets`。
- `create_analysis()`
  - 供子类覆盖。提供创建保存分析结果的结构的机会。
  - 默认行为是创建名为 `rets` 的 `OrderedDict`。
- `print(*args, **kwargs)`
  - 使用标准 `Writerfile` 对象打印 `get_analysis` 返回的结果，默认写入标准输出。
- `pprint(*args, **kwargs)`
  - 使用 Python 的 `pprint` 模块打印 `get_analysis` 返回的结果。
- `len()`
  - 支持在分析器上调用 `len`，实际上返回分析器运行的策略的当前长度。

