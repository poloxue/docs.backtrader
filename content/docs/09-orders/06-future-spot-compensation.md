---
title: "期货和现货补偿"
weight: 6
---

# 期货和现货补偿

版本 1.9.32.116 增加了对社区提出的一个有趣用例的支持：

- 用期货开仓交易，包括实物交割；
- 使用指标进行分析；
- 必要时通过操作现货价格来平仓，从而取消实物交割（收货或交货，希望能获利）；
- 期货在操作现货价格的当天到期。

这意味着：

- 平台接收两个不同资产的数据；
- 平台需要理解这些资产相关，且现货操作将关闭期货头寸；
- 实际上期货并未平仓，只是实物交割被补偿了。

基于这一补偿概念，backtrader 允许用户告知平台：一个数据流上的操作会对另一个数据流产生补偿效果。使用方式如下：

```python
import backtrader as bt

cerebro = bt.Cerebro()

data0 = bt.feeds.MyFavouriteDataFeed(dataname='futurename')
cerebro.adddata(data0)

data1 = bt.feeds.MyFavouriteDataFeed(dataname='spotname')
data1.compensate(data0)  # 告诉系统 data1 的操作会影响 data0
cerebro.adddata(data1)

...

cerebro.run()
```

## 综合示例

示例胜过千言万语，下面将所有部分结合起来。

- 使用 backtrader 源码中的标准示例数据源作为期货数据。
- 复用同一数据源，添加一个随机移动价格的过滤器来模拟价差，实现如下：

```python
# 更改收盘价的过滤器
def close_changer(data, *args, **kwargs):
    data.close[0] += 50.0 * random.randint(-1, 1)
    return False  # 数据流长度未变
```

在同一轴上绘图会混淆默认的 BuyObserver 标记，因此禁用了标准观察者，手动重新添加并使用不同的数据标记。

头寸随机进入，10 天后退出。

这并不匹配期货的到期期限，但本例仅演示功能，而非交易日历。

**注意**：
- 如需在期货到期日模拟现货价格执行，需要激活 “cheat-on-close” 确保订单在期货到期时执行。本例不需要，因为到期是随机选择的。

注意策略中的操作：

- 买入操作在 data0 上执行
- 卖出操作在 data1 上执行

```python
class St(bt.Strategy):

    def __init__(self):
        bt.obs.BuySell(self.data0, barplot=True)  # 为不同数据添加不同标记
        BuySellArrows(self.data1, barplot=True)  # 为不同数据添加不同标记

    def next(self):
        if not self.position:
            if random.randint(0, 1):
                self.buy(data=self.data0)
                self.entered = len(self)
        else:  # 在市场中
            if (len(self) - self.entered) >= 10:
                self.sell(data=self.data1)
```

### 执行：

```bash
$ ./future-spot.py --no-comp
```

得到如下图形输出。

可以看到：

- 买入操作用向上的绿色三角形标记，图例显示属于 data0
- 卖出操作用向下箭头标记，图例显示属于 data1

即使在 data0 上开仓、在 data1 上平仓，也能实现交易闭合，避免实物交割。

如果不用补偿，同样的逻辑会发生什么：

```bash
$ ./future-spot.py --no-comp
```

这会失败：

- 逻辑期望 data0 上的头寸通过 data1 的操作平仓，且仅在无持仓时才在 data0 上开仓
- 但补偿被禁用，data0 的初始操作（绿色三角形）从未平仓，无法发起其他操作，data1 上的空头头寸开始累积。

## 示例用法：

```bash
$ ./future-spot.py --help
usage: future-spot.py [-h] [--no-comp]

Compensation example

optional arguments:
  -h, --help  show this help message and exit
  --no-comp
```

## 示例代码：

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)

import argparse
import random
import backtrader as bt

# 更改收盘价的过滤器
def close_changer(data, *args, **kwargs):
    data.close[0] += 50.0 * random.randint(-1, 1)
    return False  # 数据流长度未变

# 重写标准标记
class BuySellArrows(bt.observers.BuySell):
    plotlines = dict(buy=dict(marker='$\u21E7$', markersize=12.0),
                     sell=dict(marker='$\u21E9$', markersize=12.0))

class St(bt.Strategy):
    def __init__(self):
        bt.obs.BuySell(self.data0, barplot=True)  # 为不同数据添加不同标记
        BuySellArrows(self.data1, barplot=True)  # 为不同数据添加不同标记

    def next(self):
        if not self.position:
            if random.randint(0, 1):
                self.buy(data=self.data0)
                self.entered = len(self)
        else:  # 在市场中
            if (len(self) - self.entered) >= 10:
                self.sell(data=self.data1)

def runstrat(args=None):
    args = parse_args(args)
    cerebro = bt.Cerebro()

    dataname = '../../datas/2006-day-001.txt'  # 数据源

    data0 = bt.feeds.BacktraderCSVData(dataname=dataname, name='data0')
    cerebro.adddata(data0)

    data1 = bt.feeds.BacktraderCSVData(dataname=dataname, name='data1')
    data1.addfilter(close_changer)
    if not args.no_comp:
        data1.compensate(data0)
    data1.plotinfo.plotmaster = data0
    cerebro.adddata(data1)

    cerebro.addstrategy(St)  # 示例策略

    cerebro.addobserver(bt.obs.Broker)  # 以下两行在 stdstats=False 时被移除
    cerebro.addobserver(bt.obs.Trades)

    cerebro.broker.set_coc(True)
    cerebro.run(stdstats=False)  # 执行
    cerebro.plot(volume=False)  # 绘图

def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description=('Compensation example'))

    parser.add_argument('--no-comp', required=False, action='store_true')
    return parser.parse_args(pargs)

if __name__ == '__main__':
    runstrat()
```
