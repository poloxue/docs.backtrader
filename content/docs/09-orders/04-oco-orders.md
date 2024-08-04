---
title: "OCO 订单"
weight: 4
---

# OCO订单

1.9.34.116版本增加了OCO（即One Cancel Others）到回测工具库中。

**注意**

目前仅在回测中实现，尚未对实时经纪商实现。

**注意**

在1.9.36.116版本更新中，Interactive Brokers支持StopTrail、StopTrailLimit和OCO。

使用模式尽量保持用户友好。因此，如果策略中的逻辑决定是时候发布订单，使用OCO可以像这样：

```python
def next(self):
    ...
    o1 = self.buy(...)
    ...
    o2 = self.buy(..., oco=o1)
    ...
    o3 = self.buy(..., oco=o1)  # 甚至可以是oco=o2，o2已经在o1组中
```

很简单，第一个订单o1将成为组长。通过指定oco命名参数，o2和o3成为OCO组的一部分。请注意，代码注释指出o3也可以通过指定o2成为组的一部分（o2已经是组的一部分）。

组成组后，会发生以下情况：

- 如果组中的任何订单被执行、取消或过期，其他订单将被取消。

下面的示例展示了OCO概念。一个标准执行并绘图：

```bash
$ ./oco.py --broker cash=50000 --plot
```

**注意**

现金增加到50000，因为资产价值达到4000，3个订单的1个项目至少需要12000货币单位（经纪商默认值为10000）。

以下图表实际上没有提供太多信息（这是一个标准的SMA交叉策略）。示例执行以下操作：

- 当快速SMA上穿慢速SMA时，发布3个订单。
  - order1是一个限价订单，在limdays天后到期，限价为收盘价减少一个百分比。
  - order2是一个期限更长、限价更低的限价订单。
  - order3是一个限价更低的限价订单。

因此，order2和order3不会执行，因为：

- order1将首先执行，这将触发其他订单的取消。
- 或者order1将过期，这将触发其他订单的取消。

系统保存了3个订单的ref标识符，并且只有在notify_order中看到三个ref标识符分别为Completed、Cancelled、Margin或Expired时，才会发布新买单。

退出是在持有一些酒吧后简单完成的。

为了跟踪实际执行，生成文本输出。部分内容如下：

```plaintext
2005-01-28: Oref 1 / Buy at 2941.11055
2005-01-28: Oref 2 / Buy at 2896.7722
2005-01-28: Oref 3 / Buy at 2822.87495
2005-01-31: Order ref: 1 / Type Buy / Status Submitted
2005-01-31: Order ref: 2 / Type Buy / Status Submitted
2005-01-31: Order ref: 3 / Type Buy / Status Submitted
2005-01-31: Order ref: 1 / Type Buy / Status Accepted
2005-01-31: Order ref: 2 / Type Buy / Status Accepted
2005-01-31: Order ref: 3 / Type Buy / Status Accepted
2005-02-01: Order ref: 1 / Type Buy / Status Expired
2005-02-01: Order ref: 3 / Type Buy / Status Canceled
2005-02-01: Order ref: 2 / Type Buy / Status Canceled
...
2006-06-23: Oref 49 / Buy at 3532.39925
2006-06-23: Oref 50 / Buy at 3479.147
2006-06-23: Oref 51 / Buy at 3390.39325
2006-06-26: Order ref: 49 / Type Buy / Status Submitted
2006-06-26: Order ref: 50 / Type Buy / Status Submitted
2006-06-26: Order ref: 51 / Type Buy / Status Submitted
2006-06-26: Order ref: 49 / Type Buy / Status Accepted
2006-06-26: Order ref: 50 / Type Buy / Status Accepted
2006-06-26: Order ref: 51 / Type Buy / Status Accepted
2006-06-26: Order ref: 49 / Type Buy / Status Completed
2006-06-26: Order ref: 51 / Type Buy / Status Canceled
2006-06-26: Order ref: 50 / Type Buy / Status Canceled
...
2006-11-10: Order ref: 61 / Type Buy / Status Canceled
2006-12-11: Oref 63 / Buy at 4032.62555
2006-12-11: Oref 64 / Buy at 3971.8322
2006-12-11: Oref 65 / Buy at 3870.50995
2006-12-12: Order ref: 63 / Type Buy / Status Submitted
2006-12-12: Order ref: 64 / Type Buy / Status Submitted
2006-12-12: Order ref: 65 / Type Buy / Status Submitted
2006-12-12: Order ref: 63 / Type Buy / Status Accepted
2006-12-12: Order ref: 64 / Type Buy / Status Accepted
2006-12-12: Order ref: 65 / Type Buy / Status Accepted
2006-12-15: Order ref: 63 / Type Buy / Status Expired
2006-12-15: Order ref: 65 / Type Buy / Status Canceled
2006-12-15: Order ref: 64 / Type Buy / Status Canceled
```

以下情况发生：

- 第一批订单发布。订单1到期，订单2和3被取消。预期结果。
- 几个月后发布另一批3个订单。在这种情况下，订单49完成，订单50和51立即取消。
- 最后一批订单与第一批订单相同。

现在检查不使用OCO的行为：

```bash
$ ./oco.py --strat do_oco=False --broker cash=50000
2005-01-28: Oref 1 / Buy at 2941.11055
2005-01-28: Oref 2 / Buy at 2896.7722
2005-01-28: Oref 3 / Buy at 2822.87495
2005-01-31: Order ref: 1 / Type Buy / Status Submitted
2005-01-31: Order ref: 2 / Type Buy / Status Submitted
2005-01-31: Order ref: 3 / Type Buy / Status Submitted
2005-01-31: Order ref: 1 / Type Buy / Status Accepted
2005-01-31: Order ref: 2 / Type Buy / Status Accepted
2005-01-31: Order ref: 3 / Type Buy / Status Accepted
2005-02-01: Order ref: 1 / Type Buy / Status Expired
```

结果不多（没有订单执行，图表也没有太多需求）。

发布了一批订单

订单1过期，但因为策略参数do_oco=False，订单2和3未加入OCO组

因此订单2和3未被取消，由于默认到期时间为1000天，样本可用数据（2年数据）内它们永不过期。

系统从未发布第二批订单。

#### 示例使用

```bash
$ ./oco.py --help
usage: oco.py [-h] [--data0 DATA0] [--fromdate FROMDATE] [--todate TODATE]
              [--cerebro kwargs] [--broker kwargs] [--sizer kwargs]
              [--strat kwargs] [--plot [kwargs]]

Sample Skeleton

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

### 示例代码

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse
import datetime
import backtrader as bt


class St(bt.Strategy):
    params = dict(
        ma=bt.ind.SMA,
        p1=5,
        p2=15,
        limit=0.005,
        limdays=3,
        limdays2=1000,
        hold=10,
        switchp1p2=False,  # switch prices of order1 and order2
        oco1oco2=False,

  # False - use order1 as oco for order3, else order2
        do_oco=True,  # use oco or not
    )

    def notify_order(self, order):
        print('{}: Order ref: {} / Type {} / Status {}'.format(
            self.data.datetime.date(0),
            order.ref, 'Buy' * order.isbuy() or 'Sell',
            order.getstatusname()))

        if order.status == order.Completed:
            self.holdstart = len(self)

        if not order.alive() and order.ref in self.orefs:
            self.orefs.remove(order.ref)

    def __init__(self):
        ma1, ma2 = self.p.ma(period=self.p.p1), self.p.ma(period=self.p.p2)
        self.cross = bt.ind.CrossOver(ma1, ma2)

        self.orefs = list()

    def next(self):
        if self.orefs:
            return  # pending orders do nothing

        if not self.position:
            if self.cross > 0.0:  # crossing up

                p1 = self.data.close[0] * (1.0 - self.p.limit)
                p2 = self.data.close[0] * (1.0 - 2 * 2 * self.p.limit)
                p3 = self.data.close[0] * (1.0 - 3 * 3 * self.p.limit)

                if self.p.switchp1p2:
                    p1, p2 = p2, p1

                o1 = self.buy(exectype=bt.Order.Limit, price=p1,
                              valid=datetime.timedelta(self.p.limdays))

                print('{}: Oref {} / Buy at {}'.format(
                    self.datetime.date(), o1.ref, p1))

                oco2 = o1 if self.p.do_oco else None
                o2 = self.buy(exectype=bt.Order.Limit, price=p2,
                              valid=datetime.timedelta(self.p.limdays2),
                              oco=oco2)

                print('{}: Oref {} / Buy at {}'.format(
                    self.datetime.date(), o2.ref, p2))

                if self.p.do_oco:
                    oco3 = o1 if not self.p.oco1oco2 else oco2
                else:
                    oco3 = None

                o3 = self.buy(exectype=bt.Order.Limit, price=p3,
                              valid=datetime.timedelta(self.p.limdays2),
                              oco=oco3)

                print('{}: Oref {} / Buy at {}'.format(
                    self.datetime.date(), o3.ref, p3))

                self.orefs = [o1.ref, o2.ref, o3.ref]

        else:  # in the market
            if (len(self) - self.holdstart) >= self.p.hold:
                self.close()


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
            'Sample Skeleton'
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

    parser.add_argument('--sizer', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--strat', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--plot', required=False, default='',
                        nargs='?', const='{}',
                        metavar='kwargs', help='kwargs in key=value format')

    return parser.parse_args(pargs)


if __name__ == '__main__':
    runstrat()
```
