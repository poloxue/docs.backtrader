---
title: "03. 故障排查"
weight: 3
---

### Cheat On Open（开盘作弊）

版本 1.9.44.116 添加了 Cheat-On-Open 支持。这似乎是对那些在计算完一个柱的收盘价后全仓操作的人们的需求，他们期望与开盘价匹配。

当开盘价跳空（向上或向下，取决于买入或卖出）且现金不足以进行全仓操作时，这种情况会失败。这迫使经纪商拒绝操作。

虽然人们可以尝试通过正 [1] 索引法查看未来，但这需要预加载数据，而这并不总是可行的。

### 模式：

```python
cerebro = bt.Cerebro(cheat_on_open=True)
```

这会：

- 在系统中激活一个额外的循环，该循环调用策略中的 `next_open`、`nextstart_open` 和 `prenext_open` 方法。

为了清楚地分离常规方法（这些方法基于所检查的价格不再可用且未来未知）和作弊模式下的操作，决定增加一组额外的方法家族。

这也避免了对常规 `next` 方法的两次调用。

以下情况在 `xxx_open` 方法内部保持不变：

- 指标尚未重新计算，保持上一个循环中在等效的 `xxx` 常规方法中最后看到的值。
- 经纪商尚未评估新循环中的待处理订单，并且可以引入新订单，如果可能，将进行评估。

注意：

- Cerebro 还有一个名为 `broker_coo`（默认值：True）参数，它告诉 cerebro 如果激活了 cheat-on-open，它也会尝试在经纪商中激活它（如果可能的话）。
- 模拟经纪商有一个名为 `coo` 的参数和一个名为 `set_coo` 的方法来设置它。

### 尝试 Cheat-on-open

下面的示例有一个策略，具有两种不同的行为：

- 如果 cheat-on-open 为 True，它将仅从 `next_open` 操作。
- 如果 cheat-on-open 为 False，它将仅从 `next` 操作。

在这两种情况下，匹配价格必须相同：

- 如果不作弊，订单在前一天的收盘后发出，并将与下一个到来的价格（开盘价）匹配。
- 如果作弊，订单在同一天发出并执行。因为订单是在经纪商评估订单之前发出的，所以它也将与下一个到来的价格（开盘价）匹配。

第二种情况下，可以精确计算全仓策略的份额，因为可以直接访问当前的开盘价。

在这两种情况下：

- 当前的开盘价和收盘价将从 `next` 中打印。

### 常规执行：

```bash
$ ./cheat-on-open.py --cerebro cheat_on_open=False

...
2005-04-07 next, open 3073.4 close 3090.72
2005-04-08 next, open 3092.07 close 3088.92
Strat Len 68 2005-04-08 Send Buy, fromopen False, close 3088.92
2005-04-11 Buy Executed at price 3088.47
2005-04-11 next, open 3088.47 close 3080.6
2005-04-12 next, open 3080.42 close 3065.18
...
```

订单：

- 在 2005-04-08 收盘后发出。
- 在 2005-04-11 以开盘价 3088.47 执行。

### 作弊执行：

```bash
$ ./cheat-on-open.py --cerebro cheat_on_open=True

...
2005-04-07 next, open 3073.4 close 3090.72
2005-04-08 next, open 3092.07 close 3088.92
2005-04-11 Send Buy, fromopen True, close 3080.6
2005-04-11 Buy Executed at price 3088.47
2005-04-11 next, open 3088.47 close 3080.6
2005-04-12 next, open 3080.42 close 3065.18
...
```

订单：

- 在 2005-04-11 开盘前发出。
- 在 2005-04-11 以开盘价 3088.47 执行。

### 总结
开盘作弊允许在开盘前发出订单，这可以例如允许精确计算全仓场景的份额。

### 示例用法

```bash
$ ./cheat-on-open.py --help
usage: cheat-on-open.py [-h] [--data0 DATA0] [--fromdate FROMDATE]
                        [--todate TODATE] [--cerebro kwargs] [--broker kwargs]
                        [--sizer kwargs] [--strat kwargs] [--plot [kwargs]]

Cheat-On-Open Sample

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
        periods=[10, 30],
        matype=bt.ind.SMA,
    )

    def __init__(self):
        self.cheating = self.cerebro.p.cheat_on_open
        mas = [self.p.matype(period=x) for x in self.p.periods]
        self.signal = bt.ind.CrossOver(*mas)
        self.order = None

    def notify_order(self, order):
        if order.status != order.Completed:
            return

        self.order = None
        print('{} {} Executed at price {}'.format(
            bt.num2date(order.executed.dt).date(),
            'Buy' * order.isbuy() or 'Sell', order.executed.price)
        )

    def operate(self, fromopen):
        if self.order is not None:
            return
        if self.position:
            if self.signal < 0:
                self.order = self.close()
        elif self.signal > 0:
            print('{} Send Buy, fromopen {}, close {}'.format(
                self.data.datetime.date(),
                fromopen, self.data.close[0])
            )
            self.order = self.buy()

    def next(self):
        print('{} next, open {} close {}'.format(
            self.data.datetime.date(),
            self.data.open[0], self.data.close[0])
        )

        if self.cheating:
            return
        self.operate(fromopen=False)

    def next_open(self):
        if not self.cheating:
            return
        self.operate(fromopen=True)


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
            'Cheat-On-Open Sample'
        )
    )

    parser.add_argument('--data0', default='../../datas/2005-2006-day-001.txt',
                        required=False, help='Data to read in')

    # Defaults for dates
    parser.add_argument('--fromdate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--todate', required=False

, default='',
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
