---
title: "多时间框架"
weight: 5
---

# 多时间框架策略

在实际的交易中，我们常需要结合多个时间框架来制定投资决策，如在周级别评估趋势，而在日级别执行入场，或是基于 5 分钟与 60 分钟数据的对比执行交易。在 **Backtrader** 中要实现这个目标，需要将不同时间框架的数据组合在一起。

本节将介绍如何在 **Backtrader** 实现多周期交易策略。

## 基本规则

**Backtrader** 原生支持多时间框架的数据组合，只需遵循几个简单的规则。

第一步，**最小时间框架的数据必须首先加载**。较小时间框架（条数最多的数据）应当首先加载到Cerebro实例中。

第二步，**数据必须按日期时间对齐**。为了让平台能够正确解析数据并执行策略，必须保证各时间框架的数据时间对齐。

第三步，**使用 `resampledata` 实现较大时间框架的重采样**。`cerebro.resample` 函数能轻松地将较大的时间框架数据添加到策略中。

在这个基础上，就可以在较短和较长时间框架上使用不同的技术指标。要注意，应用于大时间框架的指标产生的信号较少，还有，**Backtrader** 会考虑大时间框架的最小周期，以确保数据的准确性。

## 示例：如何使用多个时间框架

如何在 **Backtrader** 实现多时间周期呢？大概演示这个步骤吧。

### 加载数据

首先，加载较小时间框架的数据。

```python
data = btfeeds.BacktraderCSVData(dataname=datapath)
```

### 将数据添加到Cerebro

将较小时间框架数据都添加到 Cerebro 实例中。

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

首先，演示每日和每周时间框架。假设我们希望在一个策略中同时使用每日和每周的时间框架。通过命令行指定时间框架为每周，并进行数据重采样：

```bash
$ ./multitimeframe-example.py --timeframe weekly --compression 1
```

此时，程序会加载每日数据，并将其重采样为每周数据。最终输出将包括每周和每日数据的合成图表。

继续用每日时间框架压缩。如果我们希望将每日数据压缩为每两天一条数据，可以使用以下命令：

```bash
$ ./multitimeframe-example.py --timeframe daily --compression 2
```

此时，Backtrader会将每日数据压缩为每两天一条数据，并生成合成图表。

还可以带简单移动平均（SMA）指标。为了展示不同时间框架对策略的影响，可以在策略中使用简单的移动平均线（SMA）指标。SMA将在较小和较大时间框架上应用，并根据它们产生不同的信号。

- 在较小的时间框架（如每日）上，SMA将在第10个数据点后首次计算出值。
- 在较大的时间框架（如每周）上，SMA的计算会延迟，可能需要10个周期的时间来产生有效信号。

由于Backtrader的多时间框架支持，较大时间框架会消耗多个较小时间框架的数据条目来计算指标。

在策略中使用 SMA 时，如果数据点来自较大时间框架，`nextstart` 方法的调用可能会有所延迟。例如，在每周时间框架下，SMA的计算需要10周的数据，而在每个周期内，我们将看到多个“nextstart”调用，因为Backtrader会等待所有数据都齐全时才开始执行策略逻辑。

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

通过Backtrader的多时间框架支持，您可以轻松地将不同时间框架的数据结合在一起，从而实现更加灵活的交易策略。只需遵循上述规则，就能在多个时间框架中应用不同的指标，并根据数据的不同粒度调整策略的执行逻辑。

此外，Backtrader也允许通过`nextstart`方法来精确控制每个周期的数据处理逻辑，这使得您可以清晰地跟踪每个时间框架的计算过程，方便调试和优化。

