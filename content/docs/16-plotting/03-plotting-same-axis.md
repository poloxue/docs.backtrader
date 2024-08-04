---
title: "同轴绘图"
weight: 3
---

### 在同一轴上绘图

在之前的文章“future-spot”中，我们在同一空间上绘制了原始数据和略微（随机）修改后的数据，但它们并没有在同一轴上绘制。

恢复该文章的第一张图片：

![图表示例](image)

我们可以看到：

- 图表的左右两侧有不同的刻度。
- 当观察摆动的红线（随机化数据）时最明显，它在原始数据周围上下摆动大约 50 个点。
- 在图表上，视觉印象是这种随机化数据大部分时间都在原始数据上方。这只是由于不同刻度导致的视觉印象。

虽然 1.9.32.116 版本已经初步支持完全在同一轴上绘图，但图例标签会重复（仅标签，不是数据），这会让人感到困惑。

1.9.33.116 版本修复了这个问题，允许完全在同一轴上绘图。使用模式类似于选择使用哪个数据进行绘图。在之前的文章中：

```python
import backtrader as bt

cerebro = bt.Cerebro()

data0 = bt.feeds.MyFavouriteDataFeed(dataname='futurename')
cerebro.adddata(data0)

data1 = bt.feeds.MyFavouriteDataFeed(dataname='spotname')
data1.compensate(data0)  # 告诉系统 data1 的操作影响 data0
data1.plotinfo.plotmaster = data0
data1.plotinfo.sameaxis = True
cerebro.adddata(data1)

...

cerebro.run()
```

data1 获取一些 `plotinfo` 值来：

- 与 `plotmaster`（即 data0）在同一空间绘图。
- 获取使用 `sameaxis` 的指示。

原因是平台无法提前知道每个数据的刻度是否兼容。因此，它们会在独立的刻度上绘图。

在之前的示例中，增加了一个选项在 `sameaxis` 上绘图。示例执行：

```shell
$ ./future-spot.py --sameaxis
```

结果图表：

![同一轴上的绘图示例](image)

需要注意的是：

- 右侧只有一个刻度。
- 现在，随机化数据似乎明显在原始数据周围摆动，这是预期的视觉行为。

#### 示例用法

```shell
$ ./future-spot.py --help
usage: future-spot.py [-h] [--no-comp] [--sameaxis]

Compensation example

optional arguments:
  -h, --help  show this help message and exit
  --no-comp
  --sameaxis
```

#### 示例代码

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)

import argparse
import random
import backtrader as bt

# 修改收盘价的过滤器
def close_changer(data, *args, **kwargs):
    data.close[0] += 50.0 * random.randint(-1, 1)
    return False  # 流长度不变

# 重写标准标记
class BuySellArrows(bt.observers.BuySell):
    plotlines = dict(buy=dict(marker='$\u21E7$', markersize=12.0),
                     sell=dict(marker='$\u21E9$', markersize=12.0))

class St(bt.Strategy):
    def __init__(self):
        bt.obs.BuySell(self.data0, barplot=True)  # 在此处完成
        BuySellArrows(self.data1, barplot=True)  # 为不同的数据设置不同的标记

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

    dataname = '../../datas/2006-day-001.txt'  # 数据馈送

    data0 = bt.feeds.BacktraderCSVData(dataname=dataname, name='data0')
    cerebro.adddata(data0)

    data1 = bt.feeds.BacktraderCSVData(dataname=dataname, name='data1')
    data1.addfilter(close_changer)
    if not args.no_comp:
        data1.compensate(data0)
    data1.plotinfo.plotmaster = data0
    if args.sameaxis:
        data1.plotinfo.sameaxis = True
    cerebro.adddata(data1)

    cerebro.addstrategy(St)  # 示例策略

    cerebro.addobserver(bt.obs.Broker)  # 通过 stdstats=False 移除
    cerebro.addobserver(bt.obs.Trades)  # 通过 stdstats=False 移除

    cerebro.broker.set_coc(True)
    cerebro.run(stdstats=False)  # 执行
    cerebro.plot(volume=False)  # 并绘图

def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description=('Compensation example'))

    parser.add_argument('--no-comp', required=False, action='store_true')
    parser.add_argument('--sameaxis', required=False, action='store_true')
    return parser.parse_args(pargs)

if __name__ == '__main__':
    runstrat()
```


