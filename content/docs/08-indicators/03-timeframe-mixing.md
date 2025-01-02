---
title: "周期混合"
weight: 3
---

# 周期混合

如果提供值的数据源有不同的时间框架，在 `Cerebro` 引擎中有不同的长度，指标将会出错。

示例计算中，`data0` 有天的时间框架，`data1` 有月的时间框架：

```python
pivotpoint = btind.PivotPoint(self.data1)
sellsignal = self.data0.close < pivotpoint.s1
```

在这里，当收盘价低于 s1 线（第一个支撑）时，寻求卖出信号。

**注意**

`PivotPoint` 定义上在较大的时间框架中工作。

过去，这会导致以下错误：

```
return self.array[self.idx + ago]
IndexError: array index out of range
```

原因很简单：`self.data.close` 从第一个时刻提供值，但 `PivotPoint`（以及 s1 线）只有在整个月过去后才会提供值，这大约相当于 22 个 `self.data0.close` 的值。在这 22 个收盘价期间，s1 还没有值，尝试从底层数组获取它会失败。

线条对象支持 `(ago)` 操作符（Python 中的 `__call__` 特殊方法）以提供其延迟版本：

```python
close1 = self.data.close(-1)
```

在这个例子中，对象 `close1`（通过 `[0]` 访问时）始终包含由 `close` 提供的前一个值（-1）。此语法已被重用以适应时间框架。让我们重写上述的 `pivotpoint` 代码片段：

```python
pivotpoint = btind.PivotPoint(self.data1)
sellsignal = self.data0.close < pivotpoint.s1()
```

请注意，`()` 无参数执行（在后台提供了一个 `None`）。正在发生以下情况：

`pivotpoint.s1()` 返回一个内部 `LinesCoupler` 对象，该对象遵循较大范围的节奏。该耦合器使用来自实际 s1 的最新提供的值填充自身（以 `NaN` 为默认值开始）。

但为了实现这一魔法，还需要额外的东西。`Cerebro` 必须这样创建：

```python
cerebro = bt.Cerebro(runonce=False)
```

或这样执行：

```python
cerebro.run(runonce=False)
```

在这种模式下，指标和延迟评估的自动线条对象按步执行，而不是在紧密循环中执行。这使整个操作变得更慢，但它使其成为可能。

底部的示例脚本现在可以运行：

```shell
$ ./mixing-timeframes.py
```

输出：

```
0021,0021,0001,2005-01-31,2984.75,2935.96,0.00
0022,0022,0001,2005-02-01,3008.85,2935.96,0.00
...
0073,0073,0003,2005-04-15,3013.89,3010.76,0.00
0074,0074,0003,2005-04-18,2947.79,3010.76,1.00
...
```

在第 74 行，第一次出现 `close < s1`。

脚本还提供了额外的可能性：耦合一个指标的所有线条。之前是：

```python
self.sellsignal = self.data0.close < pp.s1()
```

替代方法是：

```python
pp1 = pp()
self.sellsignal = self.data0.close < pp1.s1
```

现在整个 `PivotPoint` 指标已被耦合，并且可以访问其任何线条（即 p、r1、r2、s1、s2）。脚本只对 s1 感兴趣，访问是直接的：

```shell
$ ./mixing-timeframes.py --multi
```

输出：

```
0021,0021,0001,2005-01-31,2984.75,2935.96,0.00
0022,0022,0001,2005-02-01,3008.85,2935.96,0.00
...
0073,0073,0003,2005-04-15,3013.89,3010.76,0.00
0074,0074,0003,2005-04-18,2947.79,3010.76,1.00
...
```

这里没有意外。与之前相同。“耦合”对象甚至可以绘制：

```shell
$ ./mixing-timeframes.py --multi --plot
```

### 完整耦合语法

对于具有多条线的线条对象（例如 `PivotPoint` 指标）：

```python
obj(clockref=None, line=-1)
```

`clockref`：如果 `clockref` 为 `None`，则周围对象（例如策略）将作为参考，以适应较大时间框架（例如：月）到较小/更快的时间框架（例如：日）。如果需要，可以使用另一个参考。

`line`：

- 如果默认的 `-1` 被给定，则所有线条都被耦合。
- 如果是另一个整数（例如 `0` 或 `1`），则单条线将被耦合，并通过索引（来自 `obj.lines[x]`）获取。
- 如果传递了字符串，则按名称获取线条。

在示例中，可以这样做：

```python
coupled_s1 = pp(line='s1')
```

对于只有一条线的线条对象（例如来自 `PivotPoint` 指标的 `s1` 线）：

```python
obj(clockref=None)（见上文的 `clockref`）
```

## 结论

在常规的 `()` 语法中，来自不同时间框架的数据源可以混合在指标中，始终考虑到 `cerebro` 需要使用 `runonce=False` 实例化或创建。

## 脚本代码和用法

可以在 backtrader 的源代码中找到示例。用法：

```shell
$ ./mixing-timeframes.py --help
usage: mixing-timeframes.py [-h] [--data DATA] [--multi] [--plot]

Sample for pivot point and cross plotting

optional arguments:
  -h, --help   show this help message and exit
  --data DATA  Data to be read in (default: ../../datas/2005-2006-day-001.txt)
  --multi      Couple all lines of the indicator (default: False)
  --plot       Plot the result (default: False)
```

代码：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse

import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind
import backtrader.utils.flushfile


class St(bt.Strategy):
    params = dict(multi=True)

    def __init__(self):
        self.pp = pp = btind.PivotPoint(self.data1)
        pp.plotinfo.plot = False  # deactivate plotting

        if self.p.multi:
            pp1 = pp()  # couple the entire indicators
            self.sellsignal = self.data0.close < pp1.s1
        else:
            self.sellsignal = self.data0.close < pp.s1()

    def next(self):
        txt = ','.join(
            ['%04d' % len(self),
             '%04d' % len(self.data0),
             '%04d' % len(self.data1),
             self.data.datetime.date(0).isoformat(),
             '%.2f' % self.data0.close[0],
             '%.2f' % self.pp.s1[0],
             '%.2f' % self.sellsignal[0]])

        print(txt)


def runstrat():
    args = parse_args()

    cerebro = bt.Cerebro()
    data = btfeeds.BacktraderCSVData(dataname=args.data)
    cerebro.adddata(data)
    cerebro.resampledata(data, timeframe=bt.TimeFrame.Months)

    cerebro.addstrategy(St, multi=args.multi)

    cerebro.run(stdstats=False, runonce=False)
    if args.plot:
        cerebro.plot(style='bar')


def parse_args():
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description='Sample for pivot point and cross plotting')

    parser.add_argument('--data', required=False,
                        default='../../datas/2005-2006-day-001.txt',
                        help='Data to be read in')

    parser.add_argument('--multi', required=False, action='store_true',
                        help='Couple all lines of the indicator')

    parser.add_argument('--plot', required=False, action='store_true

',
                        help=('Plot the result'))

    return parser.parse_args()


if __name__ == '__main__':
    runstrat()
```

希望这能帮助你理解如何在 backtrader 中混合不同时间框架的数据！
