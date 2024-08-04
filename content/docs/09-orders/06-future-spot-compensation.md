---
title: "期货和现货补偿"
weight: 6
---

## 期货和现货补偿

版本 1.9.32.116 增加了对社区提出的一个有趣用例的支持：

- 用期货启动交易，包括实物交割
- 使用指标进行分析
- 如有必要，通过操作现货价格来平仓，从而有效地取消实物交割，无论是收货还是交货（希望能获利）
- 期货在操作现货价格的当天到期

这意味着：

- 平台接收来自两个不同资产的数据点
- 平台必须以某种方式理解这些资产是相关的，并且现货价格的操作将关闭在期货上开启的头寸
- 实际上，期货并未平仓，只是实物交割被补偿了

利用这种补偿概念，backtrader 增加了一种方式，让用户告知平台一个数据流上的操作将对另一个数据流产生补偿效果。使用模式如下：

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

### 综合示例
一个示例胜过千言万语，所以让我们把所有的部分结合起来。

- 使用 backtrader 源代码中的一个标准示例数据源。这将是期货数据
- 通过重新使用相同的数据源并添加一个随机移动价格的过滤器来模拟一个类似但不同的价格，从而创建价差。如下简单地实现：

```python
# 更改收盘价的过滤器
def close_changer(data, *args, **kwargs):
    data.close[0] += 50.0 * random.randint(-1, 1)
    return False  # 数据流长度未变
```

在同一轴上绘图会混淆默认包含的 BuyObserver 标记，因此将禁用标准观察者并手动重新添加以使用不同的数据标记。

头寸将随机进入并在 10 天后退出。

这并不匹配期货到期期限，但这只是为了实现功能，而不是检查交易日历。

**注意**：
- 如果要在期货到期日模拟现货价格的执行，需要激活“cheat-on-close”以确保订单在期货到期时执行。这在本示例中不需要，因为到期是随机选择的。

注意策略：

- 买操作在 data0 上执行
- 卖操作在 data1 上执行

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

![图像](image)

可以看到：

- 买操作用向上的绿色三角形标记，图例显示它们属于 data0
- 卖操作用向下箭头标记，图例显示它们属于 data1

即使在 data0 上打开头寸并在 data1 上平仓，也能实现交易的闭合，达到了避免实物交割的效果。

我们可以想象，如果不进行补偿，同样的逻辑会发生什么。让我们试试：

```bash
$ ./future-spot.py --no-comp
```

输出如下：

![图像](image)

显然，这会惨败：

- 逻辑期望 data0 上的头寸通过 data1 的操作平仓，并且只有在市场不在时才在 data0 上开仓
- 但是补偿被禁用，data0 上的初始操作（绿色三角形）从未平仓，因此无法发起其他操作，data1 上的空头头寸开始累积。

### 示例用法：

```bash
$ ./future-spot.py --help
usage: future-spot.py [-h] [--no-comp]

Compensation example

optional arguments:
  -h, --help  show this help message and exit
  --no-comp
```

### 示例代码：

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
