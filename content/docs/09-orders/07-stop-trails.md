---
title: "跟踪止损（限价）"
weight: 07
---

# 跟踪止损（限价）

版本 1.9.35.116 增加了跟踪止损和跟踪止损限价订单执行类型到回测工具中。

**注意**，这只在回测中实现，尚未在实时经纪商中实现

**注意**，更新至版本 1.9.36.116。Interactive Brokers 支持跟踪止损、跟踪止损限价和 OCO。

OCO 始终将组中的第一个订单指定为参数 oco

跟踪止损限价：经纪商模拟和 IB 经纪商具有相同的行为。指定：price 作为初始止损触发价格（也指定 trailamount），然后 plimit 作为初始限价。两者之间的差值将决定 limitoffset（限价与止损触发价格之间的距离）

使用模式完全集成到策略实例的标准买、卖和平仓市场操作方法中。需要注意：

指明所需的执行类型，如 `exectype=bt.Order.StopTrail`

以及跟踪价格是否需要用固定距离或百分比距离计算

固定距离：`trailamount=10`

百分比距离：`trailpercent=0.02`（即：2%）

如果通过发出买单进入市场多头，那么带有 `StopTrail` 和 `trailamount` 的卖单会这样做：

如果未指定价格，则使用最新的收盘价

从价格中减去 trailamount 以找到止损（或触发）价格

经纪商的下一次迭代检查是否触及触发价格

如果是：订单将以市场执行类型的方式执行

如果否，使用最新的收盘价重新计算止损价格，并减去 trailamount 距离

如果新价格上涨，则更新

如果新价格下跌（或不变），则忽略

也就是说：跟踪止损价格随着价格上涨而跟随，但如果价格开始下跌则保持不变，以潜在地确保利润。

如果进入市场时发出的是卖单，那么发出带 `StopTrail` 的买单会执行相反的操作，即：价格会向下跟随。

一些使用模式

```python
# 对于向下的跟踪止损
# 将使用最后价格作为参考
self.buy(size=1, exectype=bt.Order.StopTrail, trailamount=0.25)
# 或
self.buy(size=1, exectype=bt.Order.StopTrail, price=10.50, trailamount=0.25)

# 对于向上的跟踪止损
# 将使用最后价格作为参考
self.sell(size=1, exectype=bt.Order.StopTrail, trailamount=0.25)
# 或
self.sell(size=1, exectype=bt.Order.StopTrail, price=10.50, trailamount=0.25)
```

也可以指定 `trailpercent` 而不是 `trailamount`，并将距离价格的距离计算为价格的百分比

```python
# 对于 2% 距离的向下跟踪止损
# 将使用最后价格作为参考
self.buy(size=1, exectype=bt.Order.StopTrail, trailpercent=0.02)
# 或
self.buy(size=1, exectype=bt.Order.StopTrail, price=10.50, trailpercent=0.02)

# 对于 2% 距离的向上跟踪止损
# 将使用最后价格作为参考
self.sell(size=1, exectype=bt.Order.StopTrail, trailpercent=0.02)
# 或
self.sell(size=1, exectype=bt.Order.StopTrail, price=10.50, trailpercent=0.02)
```

对于跟踪止损限价

唯一的区别在于触发跟踪止损价格时发生的事情。

在这种情况下，订单作为限价订单执行（与 `StopLimit` 订单具有相同的行为，但此时触发价格是动态的）

**注意**：必须指定 `plimit=x.x` 来买入或卖出，这将是限价

**注意**：限价不会像止损/触发价格那样动态变化

示例总是胜过千言万语，因此这里有一个 backtrader 的常规示例，它

- 使用均线向上交叉进入市场多头
- 使用跟踪止损退出市场

执行时固定价格距离为 50 点

```bash
$ ./trail.py --plot --strat trailamount=50.0
```

以及以下输出：

```plaintext
**************************************************
2005-02-14,3075.76,3025.76,3025.76
----------
2005-02-15,3086.95,3036.95,3036.95
2005-02-16,3068.55,3036.95,3018.55
2005-02-17,3067.34,3036.95,3017.34
2005-02-18,3072.04,3036.95,3022.04
2005-02-21,3063.64,3036.95,3013.64
...
...
**************************************************
2005-05-19,3051.79,3001.79,3001.79
----------
2005-05-20,3050.45,3001.79,3000.45
2005-05-23,3070.98,3020.98,3020.98
2005-05-24,3066.55,3020.98,3016.55
2005-05-25,3059.84,3020.98,3009.84
2005-05-26,3086.08,3036.08,3036.08
2005-05-27,3084.0,3036.08,3034.0
2005-05-30,3096.54,3046.54,3046.54
2005-05-31,3076.75,3046.54,3026.75
2005-06-01,3125.88,3075.88,3075.88
2005-06-02,3131.03,3081.03,3081.03
2005-06-03,3114.27,3081.03,3064.27
2005-06-06,3099.2,3081.03,3049.2
2005-06-07,3134.82,3084.82,3084.82
2005-06-08,3125.59,3084.82,3075.59
2005-06-09,3122.93,3084.82,3072.93
2005-06-10,3143.85,3093.85,3093.85
2005-06-13,3159.83,3109.83,3109.83
2005-06-14,3162.86,3112.86,3112.86
2005-06-15,3147.55,3112.86,3097.55
2005-06-16,3160.09,3112.86,3110.09
2005-06-17,3178.48,3128.48,3128.48
2005-06-20,3162.14,3128.48,3112.14
2005-06-21,3179.62,3129.62,3129.62
2005-06-22,3182.08,3132.08,3132.08
2005-06-23,3190.8,3140.8,3140.8
2005-06-24,3161.0,3140.8,3111.0
...
...
...
**************************************************
2006-12-19,4100.48,4050.48,4050.48
----------
2006-12-20,4118.54,4068.54,4068.54
2006-12-21,4112.1,4068.54,4062.1
2006-12-22,4073.5,4068.54,4023.5
2006-12-27,4134.86,4084.86,4084.86
2006-12-28,4130.66,4084.86,4080.66
2006-12-29,4119.94,4084.86,4069.94
```

而不是等待通常的向下交叉模式，系统使用跟踪止损退出市场。让我们看看第一个操作，例如：

- 进入多头时的收盘价：3075.76
- 系统计算的跟踪止损价格：3025.76（距离为 50 个单位）
- 示例计算的跟踪止损价格：3025.76（每行显示的最后一个价格）

在第一次计算之后：

- 收盘价上涨到 3086.95，止损价格调整为 3036.95
- 随后的收盘价没有超过 3086.95，触发价格不变

其他两个操作

中可以看到相同的模式。

为了比较，执行时固定距离为 30 点（仅图表）

```bash
$ ./trail.py --plot --strat trailamount=30.0
```

图表如下：

![图像](image)

最后一次执行 `trailpercent=0.02`

```bash
$ ./trail.py --plot --strat trailpercent=0.02
```

## 示例用法

```bash
$ ./trail.py --help
usage: trail.py [-h] [--data0 DATA0] [--fromdate FROMDATE] [--todate TODATE]
                [--cerebro kwargs] [--broker kwargs] [--sizer kwargs]
                [--strat kwargs] [--plot [kwargs]]

StopTrail Sample

optional arguments:
  -h, --help           show this help message and exit
  --data0 DATA0        Data to read in (default:
                       ../../datas/2005-2006-day-001.txt)
  --fromdate FROMDATE  Date[time] in YYYY-MM-DD[THH:MM:SS] format (default: )
  --todate TODATE      Date[time] in YYYY-MM-DD[THH:MM:SS] format (default: )
  --cerebro kwargs     kwargs in key=value format (default: )
  --broker kwargs      kwargs in key=value format (default: )
  --sizer kwargs       kwargs in key=value format (default: )
  --strat kwargs       kwargs in key=value format (default: )
  --plot [kwargs]      kwargs in key=value format (default: )
```

## 示例代码

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)

import argparse
import datetime
import backtrader as bt

class St(bt.Strategy):
    params = dict(
        ma=bt.ind.SMA,
        p1=10,
        p2=30,
        stoptype=bt.Order.StopTrail,
        trailamount=0.0,
        trailpercent=0.0,
    )

    def __init__(self):
        ma1, ma2 = self.p.ma(period=self.p.p1), self.p.ma(period=self.p.p2)
        self.crup = bt.ind.CrossUp(ma1, ma2)
        self.order = None

    def next(self):
        if not self.position:
            if self.crup:
                o = self.buy()
                self.order = None
                print('*' * 50)

        elif self.order is None:
            self.order = self.sell(exectype=self.p.stoptype,
                                   trailamount=self.p.trailamount,
                                   trailpercent=self.p.trailpercent)

            if self.p.trailamount:
                tcheck = self.data.close - self.p.trailamount
            else:
                tcheck = self.data.close * (1.0 - self.p.trailpercent)
            print(','.join(
                map(str, [self.datetime.date(), self.data.close[0],
                          self.order.created.price, tcheck])
                )
            )
            print('-' * 10)
        else:
            if self.p.trailamount:
                tcheck = self.data.close - self.p.trailamount
            else:
                tcheck = self.data.close * (1.0 - self.p.trailpercent)
            print(','.join(
                map(str, [self.datetime.date(), self.data.close[0],
                          self.order.created.price, tcheck])
                )
            )

def runstrat(args=None):
    args = parse_args(args)

    cerebro = bt.Cerebro()

    # Data feed kwargs
    kwargs = dict()

    # Parse from/to-date
    dtfmt, tmfmt = '%Y-%m-%d', 'T%H:%M:%S'
    for a, d in ((getattr(args, x), x) for x in ['fromdate', 'todate']):
        if a:
            strpfmt = dtfmt + tmfmt * ('T' in a)
            kwargs[d] = datetime.datetime.strptime(a, strpfmt)

    # Data feed
    data0 = bt.feeds.BacktraderCSVData(dataname=args.data0, **kwargs)
    cerebro.adddata(data0)

    # Broker
    cerebro.broker = bt.brokers.BackBroker(**eval('dict(' + args.broker + ')'))

    # Sizer
    cerebro.addsizer(bt.sizers.FixedSize, **eval('dict(' + args.sizer + ')'))

    # Strategy
    cerebro.addstrategy(St, **eval('dict(' + args.strat + ')'))

    # Execute
    cerebro.run(**eval('dict(' + args.cerebro + ')'))

    if args.plot:  # Plot if requested to
        cerebro.plot(**eval('dict(' + args.plot + ')'))

def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description=(
            'StopTrail Sample'
        )
    )

    parser.add_argument('--data0', default='../../datas/2005-2006-day-001.txt',
                        required=False, help='Data to read in')

    # Defaults for dates
    parser.add_argument('--fromdate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--todate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--cerebro', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--broker', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add.argument('--sizer', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add.argument('--strat', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add.argument('--plot', required=False, default='',
                        nargs='?', const='{}',
                        metavar='kwargs', help='kwargs in key=value format')

    return parser.parse_args(pargs)

if __name__ == '__main__':
    runstrat()
```
