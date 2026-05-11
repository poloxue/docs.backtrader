---
title: "节省内存"
weight: 3
---


# 内存优化

**Backtrader** 的开发是在大内存机器上进行的，加上绘图可视化反馈非常有用（几乎是必需品），这使得设计决策变得简单：将所有数据保存在内存中。但这一决定有一些缺点：

- 使用 `array.array` 存储数据时，超出某些边界后需要重新分配和移动数据。
- 内存较少的机器可能会受到影响。
- 连接到一个可能运行数周/月的实时数据源，提供大量秒/分钟级别的 tick 数据。

后者比前者更重要，因为backtrader做出了另一个设计决策：

- 保持纯Python以便在需要时能够在嵌入式系统中运行。

未来可能出现这样的场景：backtrader 连接到第二台机器获取实时数据，而自身运行在 Raspberry Pi 甚至资源更受限的设备上，如 ADSL 路由器（带有 Freetz 映像的 AVM Frit!Box 7490）。

因此，backtrader 需要支持动态内存管理。现在可以在实例化或运行 Cerebro 时使用以下设置：

`exactbars` 默认值为 `False`，每条线中存储的值都会保存在内存中。可能的值：

- `True`或`1`：将所有 Line 对象的内存使用减少到自动计算的最小周期。
  - 如果 SMA 的周期为30，则底层将始终有一个30条的运行缓冲区，以允许计算 SMA。
  - 此设置将停用预加载和runonce。
  - 使用此设置还将停用绘图。
  
- `-1`：在策略级别的数据源和指标/操作将保留所有数据在内存中。
  - 例如 `RSI` 内部使用指标 `UpDay` 计算，但 `UpDay` 不保留所有数据在内存中。
  - 这允许保持绘图和预加载活动。
  - runonce将被停用。

- `-2`：作为策略属性的数据源和指标将保留所有点在内存中。
  - 例如 `RSI` 内部使用指标 `UpDay` 计算，但 `UpDay` 不保留所有数据在内存中。
  - 如果在`__init__`中定义了`a = self.data.close - self.data.high`，那么`a`将不保留所有数据在内存中。
  - 这允许保持绘图和预加载活动。
  - runonce将被停用。

俗话说，百闻不如一见。下面通过示例脚本来展示差异。该脚本针对 1996 年至 2015 年的雅虎每日数据运行，共计 4965 天。

**注意**：这是一个小样本。交易 14 小时的 EuroStoxx50 期货在一个月内将生成约 18000 个 1 分钟 K 线。

首先执行脚本，查看在不请求内存节省时使用了多少内存位置：

```sh
$ ./memory-savings.py --save 0
Total memory cells used: 506430
```

对于级别1（完全节省）：

```sh
$ ./memory-savings.py --save 1
Total memory cells used: 2041
```

从 50 万降到 2041，节省效果显著。

系统中的每个 Line 对象使用 `collections.deque` 作为缓冲区（而非 `array.array`），长度限制为操作所需的最小值。

例如，使用周期为 30 的简单移动平均线的策略，会进行以下调整：

- 数据源将有一个 30 位的缓冲区，这是 SMA 生成下一个值所需的大小。
- SMA 将只有一个位置的缓冲区，因为除非有其他依赖于它的指标需要更多数据，否则无需保留更大的缓冲区。

**注意**：此模式最吸引人且可能最重要的特点是，整个脚本生命周期内使用的内存量保持不变。

无论数据源的大小如何，如果长时间连接实时数据源，这将非常有用。

但请注意：

- 绘图不可用。
- 还有其他内存消耗源，如策略生成的订单，随着时间的推移会积累。
- 此模式只能在 Cerebro 中与 `runonce=False` 配合使用。这对实时数据源也是强制性的，但在简单回测时比 `runonce=True` 慢。从某个临界点开始，内存管理的开销可能超过逐步执行回测的成本，但具体情况需要由用户自行判断。

接下来是负级别，旨在保持绘图可用的同时节省大量内存。首先是级别 -1：

```sh
$ ./memory-savings.py --save -1
Total memory cells used: 184623
```

在这种情况下，一级指标（策略中直接声明的指标）保留完整长度的缓冲区。但如果这些指标依赖于其他子指标（实际情况正是如此），则子指标的长度将受限。结果从：

- 506430 内存位置降至 184623

节省超过 50%。

**注意**：`array.array` 对象已被 `collections.deque` 替换，后者虽然内存开销更大，但操作速度更快。不过 `collections.deque` 对象本身很小，节省的内存位置大致接近估算值。

接下来是级别 -2，旨在节省策略级别声明且标记为不绘图的指标的内存：

```sh
$ ./memory-savings.py --save -2
Total memory cells used: 174695
```

现在并没有节省多少。这是因为只有一个指标被标记为不绘图：`TestInd().plotinfo.plot = False`。

让我们看看最后一个示例的绘图：

```sh
$ ./memory-savings.py --save -2 --plot
Total memory cells used: 174695
```

![](https://www.backtrader.com/docu/memory-savings/memory-savings.png)

感兴趣的读者可运行示例脚本，详细分析每个 Line 对象，遍历整个指标层次结构。启用绘图运行（级别 -1）：

```sh
$ ./memory-savings.py --save -1 --lendetails
-- Evaluating Datas
---- Data 0 Total Cells 34755 - Cells per Line 4965
-- Evaluating Indicators
---- Indicator 1.0 Average Total Cells 30 - Cells per line 30
---- SubIndicators Total Cells 1
---- Indicator 1.1 _LineDelay Total Cells 1 - Cells per line 1
---- SubIndicators Total Cells 1
...
---- Indicator 0.5 TestInd Total Cells 9930 - Cells per line 4965
---- SubIndicators Total Cells 0
-- Evaluating Observers
---- Observer 0 Total Cells 9930 - Cells per Line 4965
---- Observer 1 Total Cells 9930 - Cells per Line 4965
---- Observer 2 Total Cells 9930 - Cells per Line 4965
Total memory cells used: 184623
```

相同的，但启用最大节省（1）：

```sh
$ ./memory-savings.py --save 1 --lendetails
-- Evaluating Datas
---- Data 0 Total Cells 266 - Cells per Line 38
-- Evaluating Indicators
---- Indicator 1.0 Average Total Cells 30 - Cells per line 30
---- SubIndicators Total Cells 1
...
---- Indicator 0.5 TestInd Total Cells 2 - Cells per line 1
---- SubIndicators Total Cells 0
-- Evaluating Observers
---- Observer 0 Total Cells 2 - Cells per Line 1
---- Observer 1 Total Cells 2 - Cells per Line 1
---- Observer 2 Total Cells 2 - Cells per Line 1
```

第二个输出显示，数据源中的线已被限制为 38 个内存位置，而非 4965 个（完整数据源长度）。

并且在可能的情况下，指标和观察者已被限制为 1 个内存位置，如输出最后几行所示。

## 脚本代码和使用

在backtrader的源代码中提供了示例。使用方法：

```sh
$ ./memory-savings.py --help
usage: memory-savings.py [-h] [--data DATA] [--save SAVE] [--datalines]
                         [--lendetails] [--plot]

Check Memory Savings

optional arguments:
  -h, --help    show this help message and exit
  --data DATA   Data to be read in (default: ../../datas/yhoo-1996-2015.txt)
  --save SAVE   Memory saving level [1, 0, -1, -2] (default: 0)
  --datalines   Print data lines (default: False)
  --lendetails  Print individual items memory usage (default: False)
  --plot        Plot the result (default: False)
```

代码：

```python
from __future__ import (absolute_import

, division, print_function,
                        unicode_literals)

import argparse
import sys

import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind
import backtrader.utils.flushfile


class TestInd(bt.Indicator):
    lines = ('a', 'b')

    def __init__(self):
        self.lines.a = b = self.data.close - self.data.high
        self.lines.b = btind.SMA(b, period=20)


class St(bt.Strategy):
    params = (
        ('datalines', False),
        ('lendetails', False),
    )

    def __init__(self):
        btind.SMA()
        btind.Stochastic()
        btind.RSI()
        btind.MACD()
        btind.CCI()
        TestInd().plotinfo.plot = False

    def next(self):
        if self.p.datalines:
            txt = ','.join(
                ['%04d' % len(self),
                 '%04d' % len(self.data0),
                 self.data.datetime.date(0).isoformat()]
            )

            print(txt)

    def loglendetails(self, msg):
        if self.p.lendetails:
            print(msg)

    def stop(self):
        super(St, self).stop()

        tlen = 0
        self.loglendetails('-- Evaluating Datas')
        for i, data in enumerate(self.datas):
            tdata = 0
            for line in data.lines:
                tdata += len(line.array)
                tline = len(line.array)

            tlen += tdata
            logtxt = '---- Data {} Total Cells {} - Cells per Line {}'
            self.loglendetails(logtxt.format(i, tdata, tline))

        self.loglendetails('-- Evaluating Indicators')
        for i, ind in enumerate(self.getindicators()):
            tlen += self.rindicator(ind, i, 0)

        self.loglendetails('-- Evaluating Observers')
        for i, obs in enumerate(self.getobservers()):
            tobs = 0
            for line in obs.lines:
                tobs += len(line.array)
                tline = len(line.array)

            tlen += tdata
            logtxt = '---- Observer {} Total Cells {} - Cells per Line {}'
            self.loglendetails(logtxt.format(i, tobs, tline))

        print('Total memory cells used: {}'.format(tlen))

    def rindicator(self, ind, i, deep):
        tind = 0
        for line in ind.lines:
            tind += len(line.array)
            tline = len(line.array)

        thisind = tind

        tsub = 0
        for j, sind in enumerate(ind.getindicators()):
            tsub += self.rindicator(sind, j, deep + 1)

        iname = ind.__class__.__name__.split('.')[-1]

        logtxt = '---- Indicator {}.{} {} Total Cells {} - Cells per line {}'
        self.loglendetails(logtxt.format(deep, i, iname, tind, tline))
        logtxt = '---- SubIndicators Total Cells {}'
        self.loglendetails(logtxt.format(deep, i, iname, tsub))

        return tind + tsub


def runstrat():
    args = parse_args()

    cerebro = bt.Cerebro()
    data = btfeeds.YahooFinanceCSVData(dataname=args.data)
    cerebro.adddata(data)
    cerebro.addstrategy(
        St, datalines=args.datalines, lendetails=args.lendetails)

    cerebro.run(runonce=False, exactbars=args.save)
    if args.plot:
        cerebro.plot(style='bar')


def parse_args():
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description='Check Memory Savings')

    parser.add_argument('--data', required=False,
                        default='../../datas/yhoo-1996-2015.txt',
                        help='Data to be read in')

    parser.add_argument('--save', required=False, type=int, default=0,
                        help=('Memory saving level [1, 0, -1, -2]'))

    parser.add_argument('--datalines', required=False, action='store_true',
                        help=('Print data lines'))

    parser.add_argument('--lendetails', required=False, action='store_true',
                        help=('Print individual items memory usage'))

    parser.add_argument('--plot', required=False, action='store_true',
                        help=('Plot the result'))

    return parser.parse_args()


if __name__ == '__main__':
    runstrat()
```

