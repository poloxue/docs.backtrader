---
title: "带有信号的策略" 
weight: 2
---

### 带有信号的策略

使用 backtrader 进行操作不一定非得编写一个策略类。虽然这是首选方式，但由于对象层次结构的原因，使用信号也是可行的。

#### 快速总结：

- 不需要编写策略类、实例化指标、编写买卖逻辑等。
- 最终用户添加信号（无论如何也是指标），其余部分在后台完成。

#### 快速示例：

```python
import backtrader as bt

data = bt.feeds.OneOfTheFeeds(dataname='mydataname')
cerebro.adddata(data)

cerebro.add_signal(bt.SIGNAL_LONGSHORT, MySignal)
cerebro.run()
```

这就完成了。当然，信号本身还没有定义。让我们定义一个非常简单的信号：

- 如果收盘价高于简单移动平均线 (SMA)，则发出多头信号。
- 如果收盘价低于 SMA，则发出空头信号。

定义如下：

```python
class MySignal(bt.Indicator):
    lines = ('signal',)
    params = (('period', 30),)

    def __init__(self):
        self.lines.signal = self.data - bt.indicators.SMA(period=self.p.period)
```

现在真的完成了。当运行 `run` 时，Cerebro 会处理实例化一个特殊的策略实例，它知道如何处理这些信号。

#### 初步常见问题

**买卖操作的数量是如何确定的？**

Cerebro 实例自动为策略添加一个固定大小 (FixedSize) 的定量器。最终用户可以通过 `cerebro.addsizer` 更改定量器以改变策略。

**订单是如何执行的？**

执行类型为市价单，订单的有效期为“直到取消” (Good Until Canceled)。

#### 信号技术细节

从技术和理论角度来看，可以描述为：

- 一个可调用对象，当被调用时返回另一个对象（只调用一次）。
- 在大多数情况下，这是一个类的实例化，但不一定非得是。
- 支持 `__getitem__` 接口。唯一请求的键/索引将是 0。

从实际角度来看，信号是：

- 来自 backtrader 生态系统的 `lines` 对象，主要是指标。

这在使用其他指标时很有帮助，比如示例中的简单移动平均线。

#### 信号指示

信号在使用 `signal[0]` 查询时提供指示，含义如下：

- > 0 -> 多头指示
- < 0 -> 空头指示
- == 0 -> 无指示

示例中，简单地用 `self.data - SMA` 进行算术运算：

- 当数据高于 SMA 时发出多头指示。
- 当数据低于 SMA 时发出空头指示。

**注意**  
当未为数据指示特定价格字段时，默认参考价格为收盘价。

#### 信号类型

如下示例中的常量，直接从主 backtrader 模块获取：

```python
import backtrader as bt

bt.SIGNAL_LONG
```

有 5 种类型的信号，分为 2 组。

**主要组：**

- `LONGSHORT`：接受来自该信号的多头和空头指示。
- `LONG`：
  - 接受多头指示进行做多。
  - 接受空头指示平仓多头。但：
    - 如果系统中有 `LONGEXIT` 信号，将用它来平仓多头。
    - 如果有 `SHORT` 信号且没有 `LONGEXIT` 信号，它将被用来平仓多头再开空头。
- `SHORT`：
  - 接受空头指示进行做空。
  - 接受多头指示平仓空头。但：
    - 如果系统中有 `SHORTEXIT` 信号，将用它来平仓空头。
    - 如果有 `LONG` 信号且没有 `SHORTEXIT` 信号，它将被用来平仓空头再开多头。

**退出组：**

这两个信号旨在覆盖其他信号，并为平仓提供标准。

- `LONGEXIT`：接受空头指示平仓多头。
- `SHORTEXIT`：接受多头指示平仓空头。

#### 累积和订单并发

上面展示的示例信号会不断发出多头和空头指示，因为它只是简单地用收盘价减去 SMA 值，这总是会得到 > 0 或 < 0 的结果（0 在数学上是可能的，但实际发生的可能性很小）。

这将导致连续生成订单，从而产生两种情况：

- **累积**：即使已经在市场中，信号也会产生新订单，增加市场仓位。
- **并发**：在其他订单执行之前会生成新订单。

为了避免这种情况，默认行为是：

- 不累积。
- 不允许并发。

如果需要其中任何一种行为，可以通过 Cerebro 控制：

```python
cerebro.signal_accumulate(True)  # 或 False 禁用
cerebro.signal_concurrency(True)  # 或 False 禁用
```

#### 示例

backtrader 源代码包含一个测试功能的示例。

主要信号如下：

```python
class SMACloseSignal(bt.Indicator):
    lines = ('signal',)
    params = (('period', 30),)

    def __init__(self):
        self.lines.signal = self.data - bt.indicators.SMA(period=self.p.period)
```

如果指定了退出信号，代码如下：

```python
class SMAExitSignal(bt.Indicator):
    lines = ('signal',)
    params = (('p1', 5), ('p2', 30),)

    def __init__(self):
        sma1 = bt.indicators.SMA(period=self.p.p1)
        sma2 = bt.indicators.SMA(period=self.p.p2)
        self.lines.signal = sma1 - sma2
```

第一次运行：多头和空头

```shell
$ ./signals-strategy.py --plot --signal longshort
```

输出：

- 信号被绘制。因为它只是一个指标，所以适用绘图规则。
- 策略确实做多和做空。因为现金水平从未回到价值水平。

第二次运行：仅多头

```shell
$ ./signals-strategy.py --plot --signal longonly
```

输出：

- 现金水平在每次卖出后回到价值水平，说明策略在市场之外。

第三次运行：仅空头

```shell
$ ./signals-strategy.py --plot --signal shortonly
```

输出：

- 第一次操作是卖出，发生在收盘价低于 SMA 且简单减法得到负值之后。
- 现金水平在每次买入后回到价值水平，说明策略在市场之外。

第四次运行：多头 + 多头退出

```shell
$ ./signals-strategy.py --plot --signal longonly --exitsignal longexit
```

输出：

- 许多交易是相同的，但一些较早被中断，因为退出信号中的快速移动平均线向下穿过慢速移动平均线。
- 系统显示其仅做多特性，每笔交易结束时现金成为价值。

#### 使用方法

```shell
$ ./signals-strategy.py --help
```

使用示例：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse
import collections
import datetime

import backtrader as bt

MAINSIGNALS = collections.OrderedDict(
    (('longshort', bt.SIGNAL_LONGSHORT),
     ('longonly', bt.SIGNAL_LONG),
     ('shortonly', bt.SIGNAL_SHORT),)
)


EXITSIGNALS = {
    'longexit': bt.SIGNAL_LONGEXIT,
    'shortexit': bt.SIGNAL_LONGEXIT,
}


class SMACloseSignal(bt.Indicator):
    lines = ('signal',)
    params = (('period', 30),)

    def __init__(self):
        self.lines.signal = self.data - bt.indicators.SMA(period=self.p.period)


class SMAExitSignal(bt.Indicator):
    lines = ('signal',)
    params = (('p1', 5), ('p2', 30),)

    def __init__(self):
        sma1 = bt.indicators.SMA(period=self.p.p1)
        sma2 = bt.indicators.SMA(period=self.p.p2)
        self.lines.signal = sma1 - sma2


def runstrat(args=None):
    args = parse_args(args)

    cerebro = bt.Cerebro()
    cerebro.broker.set_cash(args.cash)

    dkwargs = dict()
    if args.fromdate is not None:
        fromdate = datetime.datetime.strptime(args.fromdate, '%Y-%m-%d')
        dkwargs['fromdate'] = fromdate

    if args.todate is not None:
        todate = datetime.datetime.strptime(args.todate, '%Y-%m-%d')
        dkwargs['todate'] = todate

    data = bt.feeds.BacktraderCSVData(dataname=args.data, **dkwargs)
    cerebro.adddata(data)

    cerebro.add_signal(MAINSIGNALS[args.signal],
                       SMACloseSignal, period=args.smaperiod)

    if args.ex

itsignal is not None:
        cerebro.add_signal(EXITSIGNALS[args.exitsignal],
                           SMAExitSignal,
                           p1=args.exitperiod,
                           p2=args.smaperiod)

    cerebro.run()
    if args.plot:
        pkwargs = dict(style='bar')
        if args.plot is not True:
            npkwargs = eval('dict(' + args.plot + ')')
            pkwargs.update(npkwargs)

        cerebro.plot(**pkwargs)


def parse_args(pargs=None):

    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description='Sample for Signal concepts')

    parser.add_argument('--data', required=False,
                        default='../../datas/2005-2006-day-001.txt',
                        help='Specific data to be read in')

    parser.add_argument('--fromdate', required=False, default=None,
                        help='Starting date in YYYY-MM-DD format')

    parser.add_argument('--todate', required=False, default=None,
                        help='Ending date in YYYY-MM-DD format')

    parser.add_argument('--cash', required=False, action='store',
                        type=float, default=50000,
                        help=('Cash to start with'))

    parser.add_argument('--smaperiod', required=False, action='store',
                        type=int, default=30,
                        help=('Period for the moving average'))

    parser.add_argument('--exitperiod', required=False, action='store',
                        type=int, default=5,
                        help=('Period for the exit control SMA'))

    parser.add_argument('--signal', required=False, action='store',
                        default=MAINSIGNALS.keys()[0], choices=MAINSIGNALS,
                        help=('Signal type to use for the main signal'))

    parser.add_argument('--exitsignal', required=False, action='store',
                        default=None, choices=EXITSIGNALS,
                        help=('Signal type to use for the exit signal'))

    parser.add_argument('--plot', '-p', nargs='?', required=False,
                        metavar='kwargs', const=True,
                        help=('Plot the read data applying any kwargs passed\n'
                              '\n'
                              'For example:\n'
                              '\n'
                              '  --plot style="candle" (to plot candles)\n'))

    if pargs is not None:
        return parser.parse_args(pargs)

    return parser.parse_args()


if __name__ == '__main__':
    runstrat()
```
