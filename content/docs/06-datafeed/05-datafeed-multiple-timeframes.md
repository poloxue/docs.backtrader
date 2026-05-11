---
title: "多时间框架"
weight: 5
---

# 多时间框架策略

实际交易中常需要结合多个时间框架来制定决策，如在周线评估趋势、在日线执行入场，或基于 5 分钟与 60 分钟数据的对比进行交易。在 **Backtrader** 中需要将不同时间框架的数据组合在一起。

本节介绍如何在 **Backtrader** 中实现多周期策略。

## 基本规则

**Backtrader** 原生支持多时间框架的数据组合，只需遵循几个简单的规则。

第一步，**最小时间框架的数据必须最先加载**。较小时间框架（数据条数最多）应首先加载到 Cerebro 实例中。

第二步，**数据必须按日期时间对齐**，以确保平台能正确解析并执行策略。

第三步，**使用 `resampledata` 将数据重采样到较大时间框架**。`cerebro.resample` 函数可以轻松实现。

在此基础上，可以在较短和较长时间框架上使用不同的指标。注意，大时间框架的指标产生的信号较少，且 **Backtrader** 会考虑大时间框架的最小周期以确保数据准确性。

## 示例：如何使用多个时间框架

下面演示在 **Backtrader** 中实现多时间周期的步骤。

### 加载数据

首先，加载较小时间框架的数据。

```python
data = btfeeds.BacktraderCSVData(dataname=datapath)
```

### 将数据添加到Cerebro

将较小时间框架数据添加到 Cerebro 实例中。

```python
cerebro.adddata(data)
```

### 重采样数据

使用 `cerebro.resampledata` 将数据重采样到较大的时间框架。

```python
cerebro.resampledata(data, timeframe=tframes[args.timeframe], compression=args.compression)
```

### 运行策略

执行策略并生成结果。

```python
cerebro.run()
```

### 示例

先演示每日和每周时间框架。假设要在一个策略中同时使用每日和每周数据，通过命令行指定时间框架为每周并进行重采样：

```bash
$ ./multitimeframe-example.py --timeframe weekly --compression 1
```

此时，程序会加载每日数据，并将其重采样为每周数据。最终输出将包括每周和每日数据的合成图表。

继续用每日时间框架压缩。如果希望将每日数据压缩为每两天一条，可以使用以下命令：

```bash
$ ./multitimeframe-example.py --timeframe daily --compression 2
```

此时，Backtrader会将每日数据压缩为每两天一条数据，并生成合成图表。

还可以加入 SMA 指标来展示不同时间框架的影响。SMA 将在大小时间框架上分别应用，产生不同的信号。

- 在较小时间框架（如每日）上，SMA 在第 10 个数据点后首次计算。
- 在较大时间框架（如每周）上，SMA 的计算会延迟，需要 10 个周期才产生有效信号。

由于 Backtrader 的多时间框架支持，较大时间框架会消耗多个较小时间框架的数据条目来计算指标。

使用 SMA 时，如果数据点来自较大时间框架，`nextstart` 方法的调用可能延迟。例如在每周框架下，SMA 需要 10 周的数据，过程中会多次触发 `nextstart`，因为 Backtrader 会等待所有数据齐全后才执行策略逻辑。

## 代码示例

```python
# 导入必要的库
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse
import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind

# 创建SMA策略
class SMAStrategy(bt.Strategy):
    params = (
        ('period', 10),  # SMA的周期
        ('onlydaily', False),  # 是否只在每日时间框架上应用
    )

    def __init__(self):
        # 为较小时间框架添加SMA
        self.sma_small_tf = btind.SMA(self.data, period=self.p.period)
        
        # 如果选择不只应用于每日时间框架
        if not self.p.onlydaily:
            # 为较大时间框架（如每周）添加SMA
            self.sma_large_tf = btind.SMA(self.data1, period=self.p.period)

    # nextstart方法，用于输出调试信息
    def nextstart(self):
        print('--------------------------------------------------')
        print('nextstart called with len', len(self))
        print('--------------------------------------------------')
        super(SMAStrategy, self).nextstart()

# 运行策略
def runstrat():
    args = parse_args()

    # 创建Cerebro实例
    cerebro = bt.Cerebro(stdstats=False)

    # 根据用户选择的策略参数加载相应策略
    if not args.indicators:
        cerebro.addstrategy(bt.Strategy)
    else:
        cerebro.addstrategy(SMAStrategy, period=args.period, onlydaily=args.onlydaily)

    # 加载数据文件
    datapath = args.dataname or '../../datas/2006-day-001.txt'
    data = btfeeds.BacktraderCSVData(dataname=datapath)
    cerebro.adddata(data)  # 添加较小时间框架的数据

    tframes = dict(daily=bt.TimeFrame.Days, weekly=bt.TimeFrame.Weeks, monthly=bt.TimeFrame.Months)

    # 根据需要重采样数据到较大时间框架
    if args.noresample:
        datapath = args.dataname2 or '../../datas/2006-week-001.txt'
        data2 = btfeeds.BacktraderCSVData(dataname=datapath)
        cerebro.adddata(data2)
    else:
        cerebro.resampledata(data, timeframe=tframes[args.timeframe], compression=args.compression)

    # 执行策略并生成结果
    cerebro.run()

    # 绘制结果
    cerebro.plot(style='bar')

# 解析命令行参数
def parse_args():
    parser = argparse.ArgumentParser(description='Multitimeframe test')
    parser.add_argument('--dataname', default='', required=False, help='数据文件路径')
    parser.add_argument('--dataname2', default='', required=False, help='第二个数据文件路径')
    parser.add_argument('--noresample', action='store_true', help='不进行数据重采样')
    parser.add_argument('--timeframe', default='weekly', choices=['daily', 'weekly', 'monthly'], help='重采样时间框架')
    parser.add_argument('--compression', default=1, type=int, help='压缩数据条数')
    parser.add_argument('--indicators', action='store_true', help='是否使用带指标的策略')
    parser.add_argument('--onlydaily', action='store_true', help='仅在每日时间框架上应用指标')
    parser.add_argument('--period', default=10, type=int, help='指标周期')
    return parser.parse_args()

if __name__ == '__main__':
    runstrat()
```

## 结论

Backtrader 的多时间框架支持让你轻松结合不同时间框架的数据，实现更灵活的交易策略。按照上述规则，即可在多个时间框架上应用不同指标，并调整策略执行逻辑。

此外，Backtrader 还提供 `nextstart` 方法精确控制每个周期的数据，方便跟踪和调试。

