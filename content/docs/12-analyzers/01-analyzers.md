---
title: "Analyzers"
weight: 1
---

# 分析器

无论是回测还是交易，能够分析交易系统的表现对于理解是否不仅仅获得了利润，而且是否在实现利润的过程中承担了过多的风险，或者与参考资产（或无风险资产）相比是否真的值得付出努力，都是关键。

这就是分析器对象的作用：提供对已发生情况或当前情况的分析。

分析器的性质
接口的设计参照了线条对象，例如具有 next 方法，但有一个主要的区别：分析器不包含线条。

这意味着它们在内存方面不是很消耗资源，因为即使在分析了成千上万个价格条之后，它们可能仍然只在内存中保存单个结果。

## 在生态系统中的位置

分析器对象（如同策略、观察者和数据）通过 cerebro 实例添加到系统中：

```python
addanalyzer(ancls, *args, **kwargs)
```

但在 cerebro.run 操作期间，对于系统中存在的每个策略，将会发生以下情况：

- ancls 在 cerebro.run 期间会用 *args 和 **kwargs 实例化
- ancls 实例将会附加到策略上

这意味着：

如果回测运行包含例如 3 个策略，那么将创建 3 个 ancls 实例，并且每个实例将附加到不同的策略上。

结论：分析器分析单个策略的表现，而不是整个系统的表现。

## 附加位置

某些分析器对象可能实际上使用其他分析器来完成其工作。例如：SharpeRatio 使用 TimeReturn 的输出进行计算。

这些子分析器或从属分析器也将插入到创建它们的同一策略中。但它们对用户是完全不可见的。

## 属性

为了完成预期的工作，分析器对象提供了一些默认属性，这些属性会自动传递并在实例中设置，以便于使用：

- `self.strategy`：对分析器对象正在运行的策略子类的引用。策略可以访问的任何内容，分析器也可以访问。
- `self.datas[x]`：策略中存在的数据源数组。虽然可以通过策略引用访问，但快捷方式使工作更舒适。
- `self.data`：对 `self.datas[0]` 的快捷方式，以增加舒适度。
- `self.dataX`：对不同的 `self.datas[x]` 的快捷方式。

还有一些其他别名可用，尽管它们可能有些多余：

- `self.dataX_Y`，其中 X 是对 `self.datas[X]` 的引用，Y 指的是线条，最终指向 `self.datas[X].lines[Y]`。
如果线条有名称，还可以使用以下命名：

- `self.dataX_Name`，解析为 `self.datas[X].Name`，按名称而不是按索引返回线条。
对于第一个数据，最后两个快捷方式在没有初始 X 数字引用的情况下可用。例如：

- `self.data_2` 指 `self.datas[0].lines[2]`。
- `self.data_close` 指 `self.datas[0].close`。

## 返回分析结果

Analyzer 基类创建了一个 `self.rets`（类型为 collections.OrderedDict）成员属性来返回分析结果。这是在 `create_analysis` 方法中完成的，子类可以覆盖该方法以创建自定义分析器。

## 操作模式

虽然分析器对象不是线条对象，因此不迭代线条，但它们被设计为遵循相同的操作模式。

- 在系统启动之前实例化（因此调用 `__init__`）。
- 使用 `start` 信号表示操作开始。
- `prenext` / `nextstart` / `next` 方法将根据策略的计算最小周期被调用。
- `prenext` 和 `nextstart` 的默认行为是调用 `next`，因为分析器可能会从系统开始运行的第一刻起进行分析。
- 在线条对象中通常调用 `len(self)` 以检查实际条数。这在分析器中也适用，通过返回 `self.strategy` 的值。
- 订单和交易将像通知策略一样通过 `notify_order` 和 `notify_trade` 进行通知。
- 现金和价值也将通过 `notify_cashvalue` 方法像对策略那样进行通知。
- 现金、价值、基金价值和基金份额也将通过 `notify_fund` 方法像对策略那样进行通知。
- 使用 `stop` 信号表示操作结束。
- 一旦常规操作周期完成，分析器将提供用于提取/输出信息的附加方法：
  - `get_analysis`：理想情况下（不是强制的）返回一个包含分析结果的字典样对象。
  - `print` 使用标准的 backtrader.WriterFile（除非被覆盖）写出 `get_analysis` 的分析结果。
  - `pprint`（漂亮打印）使用 Python 的 pprint 模块打印 `get_analysis` 结果。

最后：

`get_analysis` 创建一个类型为 collections.OrderedDict 的成员属性 `self.ret`，分析器将分析结果写入其中。

分析器的子类可以覆盖此方法以改变此行为。

## 分析器模式

在 backtrader 平台中开发分析器对象揭示了生成分析的两种不同使用模式：

- 在执行过程中通过 `notify_xxx` 和 `next` 方法收集信息，并在 `next` 中生成当前的分析信息。
  - 例如，TradeAnalyzer 仅使用 `notify_trade` 方法生成统计数据。
- 如上所述收集信息（或不收集），但在 `stop` 方法期间进行单次生成分析。
  - SQN（系统质量编号）在 `notify_trade` 期间收集交易信息，但在 `stop` 方法期间生成统计数据。

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

没有绘图，因为 SharpeRatio 是在计算结束时的单一值。

分析器的取证分析
让我们重复一下，分析器不是线条对象，但为了无缝集成到 backtrader 生态系统中，遵循了几个线条对象的内部 API 约定（实际上是它们的混合）。

注意

SharpeRatio 的代码已经演变，例如考虑了年化，此版本仅应作为参考。

请查看分析器参考。

此外还有一个 SharpeRatio_A 提供了直接以年化形式给出的值，而不管所需的时间范围。

SharpeRatio 的代码作为基础（简化版本）：

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

代码可以分解为：

- `params` 声明
  - 尽管声明的参数没有使用（作为示例），分析器像大多数其他对象一样支持参数。
- `__init__` 方法
  - 就像策略在 `__init__` 中声明指标一样，分析器也声明支持对象。
  - 在这种情况下：SharpeRatio 使用年度回报计算。计算将是自动的，并且将可用于 SharpeRatio 进行自己的计算。
- `next` 方法
  - SharpeRatio 不需要它，但此方法将在父策略 `next` 的每次调用后被调用。
- `start` 方法
  - 在回测开始前调用。可以用于额外的初始化任务。

SharpeRatio 不需要它。
- `stop` 方法
  - 在回测结束后调用。就像 SharpeRatio 所做的那样，它可以用于完成/进行计算。
- `get_analysis` 方法（返回字典）
  - 为外部调用者提供生成的分析。

返回：包含分析结果的字典。

## 参考

```python
class backtrader.Analyzer()
```

Analyzer 基类。所有分析器都是此类的子类。

分析器实例在策略的框架内运行，并为该策略提供分析。

自动设置的成员属性：

- `self.strategy`（提供对策略及其可访问内容的访问）
- `self.datas[x]` 提供对系统中存在的数据源数组的访问，这些数据源也可以通过策略引用访问
- `self.data` 提供对 `self.datas[0]` 的访问
- `self.dataX` -> `self.datas[X]`
- `self.dataX_Y` -> `self.datas[X].lines[Y]`
- `self.dataX_name` -> `self.datas[X].name`
- `self.data_name` -> `self.datas[0].name`
- `self.data_Y` -> `self.datas[0].lines[Y]`

这不是线条对象，但方法和操作遵循相同的设计：

- 在实例化和初始设置期间 `__init__`
- 使用 `start` / `stop` 表示操作的开始和结束
- `prenext` / `nextstart` / `next` 方法家族，遵循策略中相同方法的调用
- `notify_trade` / `notify_order` / `notify_cashvalue` / `notify_fund` 接收与策略等效方法相同的通知

操作模式是开放的，没有首选模式。因此，分析可以通过 `next` 调用生成，在 `stop` 期间生成，也可以通过单个方法如 `notify_trade` 生成。

重要的是覆盖 `get_analysis` 以返回包含分析结果的字典样对象（实际格式取决于实现）。

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

