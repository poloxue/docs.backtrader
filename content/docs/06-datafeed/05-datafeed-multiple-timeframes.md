---
title: "数据 - 多时间框架"
weight: 5
---

### 数据 - 多时间框架

有时，投资决策需要使用不同的时间框架：

- 每周评估趋势
- 每日执行入场
- 或5分钟与60分钟对比

这意味着需要在backtrader中组合多个时间框架的数据，以支持这种组合。

原生支持已内置。最终用户只需遵循以下规则：

1. 最小时间框架（即条数最多）的数据必须首先添加到Cerebro实例中。
2. 数据必须正确地日期时间对齐，使平台能够解析它们。

除此之外，用户可以随意在较短/较长的时间框架上应用指标。当然：

- 应用于较大时间框架的指标会产生较少的条数。
- 平台还会考虑到较大时间框架的最小周期。

最小周期可能会导致在Cerebro中添加的策略在开始执行之前，需要消耗几个数量级的较小时间框架的条数。

将使用内置的`cerebro.resample`来创建较大的时间框架。

下面是一些示例，但首先是测试脚本的基础代码。

```python
# 加载数据
datapath = args.dataname or '../../datas/2006-day-001.txt'
data = btfeeds.BacktraderCSVData(dataname=datapath)
cerebro.adddata(data)  # 首先添加原始数据 - 较小的时间框架

tframes = dict(daily=bt.TimeFrame.Days, weekly=bt.TimeFrame.Weeks,
               monthly=bt.TimeFrame.Months)

# 方便的字典用于时间框架参数转换
# 重采样数据
if args.noresample:
    datapath = args.dataname2 or '../../datas/2006-week-001.txt'
    data2 = btfeeds.BacktraderCSVData(dataname=datapath)
    # 然后是较大的时间框架
    cerebro.adddata(data2)
else:
    cerebro.resampledata(data, timeframe=tframes[args.timeframe],
                         compression=args.compression)

# 运行所有内容
cerebro.run()
```

### 步骤：

1. 加载数据
2. 根据用户指定的参数重采样
3. 脚本还允许加载第二个数据
4. 将数据添加到cerebro
5. 将重采样的数据（较大时间框架）添加到cerebro
6. 运行

#### 示例 1 - 每日和每周

脚本调用：

```bash
$ ./multitimeframe-example.py --timeframe weekly --compression 1
```

输出图表：

（此处应有图表）

#### 示例 2 - 每日和每日压缩（2条变1条）

脚本调用：

```bash
$ ./multitimeframe-example.py --timeframe daily --compression 2
```

输出图表：

（此处应有图表）

#### 示例 3 - 带有SMA的策略

虽然绘图很有用，但关键问题是展示较大时间框架如何影响系统，特别是涉及到起始点时。

脚本可以使用`--indicators`参数添加一个创建10周期简单移动平均线的策略，应用于较小和较大时间框架数据。

如果只考虑较小的时间框架：

- 在第10个条之后会首次调用`next`，这是简单移动平均线需要的时间来产生一个值。

注意：策略监视创建的指标，并且只有在所有指标产生值时才调用`next`。其逻辑是，用户添加指标是为了在逻辑中使用它们，因此如果指标没有产生值，就不应进行任何逻辑操作。

但在这种情况下，较大时间框架（每周）延迟了`next`的调用，直到较大时间框架上的简单移动平均线产生一个值，这需要……10周。

脚本重写了`nextstart`，它只调用一次，默认调用`next`来显示首次调用的时间。

调用1：
只有较小的时间框架（每日）得到一个简单移动平均线

命令行和输出：

```bash
$ ./multitimeframe-example.py --timeframe weekly --compression 1 --indicators --onlydaily
--------------------------------------------------
nextstart called with len 10
--------------------------------------------------
```

输出图表：

（此处应有图表）

调用2：
两个时间框架都得到一个简单移动平均线

命令行：

```bash
$ ./multitimeframe-example.py --timeframe weekly --compression 1 --indicators
--------------------------------------------------
nextstart called with len 50
--------------------------------------------------
--------------------------------------------------
nextstart called with len 51
--------------------------------------------------
--------------------------------------------------
nextstart called with len 52
--------------------------------------------------
--------------------------------------------------
nextstart called with len 53
--------------------------------------------------
--------------------------------------------------
nextstart called with len 54
--------------------------------------------------
```

需要注意的两点：

1. 策略在50个周期后首次调用，而不是10个周期后。
2. 这是因为应用于较大时间框架（每周）的简单移动平均线在10周后产生一个值，而这相当于10周 * 5天/周……50天。

`nextstart`被调用了5次，而不是只调用一次。这是混合时间框架并应用于较大时间框架的指标的自然副作用。较大时间框架的简单移动平均线在消耗5个每日条时会产生5次相同的值。因此，`nextstart`被调用了5次。

输出图表：

（此处应有图表）

### 结论

在backtrader中使用多个时间框架数据不需要特殊的对象或调整：只需首先添加较小的时间框架。

### 测试脚本

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse

import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind


class SMAStrategy(bt.Strategy):
    params = (
        ('period', 10),
        ('onlydaily', False),
    )

    def __init__(self):
        self.sma_small_tf = btind.SMA(self.data, period=self.p.period)
        if not self.p.onlydaily:
            self.sma_large_tf = btind.SMA(self.data1, period=self.p.period)

    def nextstart(self):
        print('--------------------------------------------------')
        print('nextstart called with len', len(self))
        print('--------------------------------------------------')

        super(SMAStrategy, self).nextstart()


def runstrat():
    args = parse_args()

    # 创建 cerebro 实体
    cerebro = bt.Cerebro(stdstats=False)

    # 添加策略
    if not args.indicators:
        cerebro.addstrategy(bt.Strategy)
    else:
        cerebro.addstrategy(
            SMAStrategy,

            # 策略参数
            period=args.period,
            onlydaily=args.onlydaily,
        )

    # 加载数据
    datapath = args.dataname or '../../datas/2006-day-001.txt'
    data = btfeeds.BacktraderCSVData(dataname=datapath)
    cerebro.adddata(data)  # 首先添加原始数据 - 较小的时间框架

    tframes = dict(daily=bt.TimeFrame.Days, weekly=bt.TimeFrame.Weeks,
                   monthly=bt.TimeFrame.Months)

    # 方便的字典用于时间框架参数转换
    # 重采样数据
    if args.noresample:
        datapath = args.dataname2 or '../../datas/2006-week-001.txt'
        data2 = btfeeds.BacktraderCSVData(dataname=datapath)
        # 然后是较大的时间框架
        cerebro.adddata(data2)
    else:
        cerebro.resampledata(data, timeframe=tframes[args.timeframe],
                             compression=args.compression)

    # 运行所有内容
    cerebro.run()

    # 绘制结果
    cerebro.plot(style='bar')


def parse_args():
    parser = argparse.ArgumentParser(
        description='Multitimeframe test')

    parser.add_argument('--dataname', default='', required=False,
                        help='文件数据加载路径')

    parser.add_argument('--dataname2', default='', required=False,
                        help='加载较大时间框架文件路径')

    parser.add_argument('--noresample', action='store_true',
                        help='不重采样，而是加载较大时间框架')

    parser.add_argument('--timeframe', default='weekly', required=False,
                        choices=['daily', 'weekly', 'monthly'],
                        help='要重采样到的时间框架')

    parser.add_argument('--compression', default=1, required=False, type=int,
                        help='压缩n条数据为1条')

    parser.add_argument('--indicators', action='storetrue',
                        help='是否应用带指标的策略')

    parser.add_argument('--onlydaily', action='storetrue',
                        help='仅将指标应用于每日时间框架')

    parser.add_argument('--period', default=10, required=False, type=int,
                        help='应用于指标的周期')

    return parser.parse_args()


if __name__ == '__main__':
    runstrat()
```
