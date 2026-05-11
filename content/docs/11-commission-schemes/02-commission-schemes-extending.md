---
title: "扩展"
weight: 2
---

# 扩展佣金

佣金及相关功能由一个单一的类 `CommissionInfo` 管理，通常通过调用 `broker.setcommission` 实例化。

这个概念仅限于带有保证金和每份合约固定佣金的期货，以及基于价格/数量百分比佣金的股票。虽然不是最灵活的方案，但它已经发挥了作用。

GitHub 上的一个增强请求（#29）导致了一些重构，以便：

- 保持 `CommissionInfo` 和 `broker.setcommission` 与原始行为兼容
- 清理一些代码
- 使佣金方案更灵活，以支持增强需求和更多可能性

实际工作如下：

```python
class CommInfoBase(with_metaclass(MetaParams)):
    COMM_PERC, COMM_FIXED = range(2)

    params = (
        ('commission', 0.0), ('mult', 1.0), ('margin', None),
        ('commtype', None),
        ('stocklike', False),
        ('percabs', False),
    )
```

引入了 `CommissionInfo` 的基类，新增了以下参数：

- `commtype`（默认值：None）

  这是兼容性的关键。如果值为 None，`CommissionInfo` 对象和 `broker.setcommission` 的行为与以前相同。具体如下：

  - 如果设置了 `margin`，则佣金方案是期货，每份合约固定佣金
  - 如果未设置 `margin`，则佣金方案是股票，采用百分比方式
  - 如果值为 `COMM_PERC` 或 `COMM_FIXED`（或派生类中的任何其他值），则决定佣金是固定还是基于百分比

- `stocklike`（默认值：False）

  如上所述，旧 `CommissionInfo` 对象中的实际行为由 `margin` 参数决定。

  如果 `commtype` 设置为其他值，则此参数指示资产是类似期货（使用保证金并进行基于柱状图的现金调整）还是类似股票。

- `percabs`（默认值：False）

  如果为 False，百分比以相对值传递（xx%）

  如果为 True，百分比以绝对值传递（0.xx）

`CommissionInfo` 从 `CommInfoBase` 派生，将此参数的默认值改为 True 以保持兼容行为。

所有这些参数也可以在 `broker.setcommission` 中使用，现在看起来像这样：

```python
def setcommission(self,
                  commission=0.0, margin=None, mult=1.0,
                  commtype=None, percabs=True, stocklike=False,
                  name=None):
```

注意：

- `percabs` 为 True，以保持与 `CommissionInfo` 旧调用的兼容性

旧的测试佣金方案示例已重写，支持命令行参数和新行为。帮助信息如下：

```shell
$ ./commission-schemes.py --help
usage: commission-schemes.py [-h] [--data DATA] [--fromdate FROMDATE]
                             [--todate TODATE] [--stake STAKE]
                             [--period PERIOD] [--cash CASH] [--comm COMM]
                             [--mult MULT] [--margin MARGIN]
                             [--commtype {none,perc,fixed}] [--stocklike]
                             [--percrel] [--plot] [--numfigs NUMFIGS]

Commission schemes

optional arguments:
  -h, --help            show this help message and exit
  --data DATA, -d DATA  data to add to the system (default:
                        ../../datas/2006-day-001.txt)
  --fromdate FROMDATE, -f FROMDATE
                        Starting date in YYYY-MM-DD format (default:
                        2006-01-01)
  --todate TODATE, -t TODATE
                        Starting date in YYYY-MM-DD format (default:
                        2006-12-31)
  --stake STAKE         Stake to apply in each operation (default: 1)
  --period PERIOD       Period to apply to the Simple Moving Average (default:
                        30)
  --cash CASH           Starting Cash (default: 10000.0)
  --comm COMM           Commission factor for operation, either apercentage or
                        a per stake unit absolute value (default: 2.0)
  --mult MULT           Multiplier for operations calculation (default: 10)
  --margin MARGIN       Margin for futures-like operations (default: 2000.0)
  --commtype {none,perc,fixed}
                        Commission - choose none for the old CommissionInfo
                        behavior (default: none)
  --stocklike           If the operation is for stock-like assets orfuture-
                        like assets (default: False)
  --percrel             If perc is expressed in relative xx% ratherthan
                        absolute value 0.xx (default: False)
  --plot, -p            Plot the read data (default: False)
  --numfigs NUMFIGS, -n NUMFIGS
                        Plot using numfigs figures (default: 1)
```

让我们运行几个示例来重现原始佣金方案的行为。

## 期货佣金（固定且带保证金）

执行和图表：

```shell
$ ./commission-schemes.py --comm 2.0 --margin 2000.0 --mult 10 --plot
```

输出显示固定佣金为 2.0 货币单位（默认 `stake` 为 1）：

```plaintext
2006-03-09, BUY CREATE, 3757.59
2006-03-10, BUY EXECUTED, Price: 3754.13, Cost: 2000.00, Comm 2.00
2006-04-11, SELL CREATE, 3788.81
2006-04-12, SELL EXECUTED, Price: 3786.93, Cost: 2000.00, Comm 2.00
2006-04-12, TRADE PROFIT, GROSS 328.00, NET 324.00
...
```

## 股票佣金（百分比且无保证金）

执行和图表：

```shell
$ ./commission-schemes.py --comm 0.005 --margin 0 --mult 1 --plot
```

为了更直观，可以使用相对百分比值：

```shell
$ ./commission-schemes.py --percrel --comm 0.5 --margin 0 --mult 1 --plot
```

现在 0.5 直接表示 0.5%

输出在两种情况下都是：

```plaintext
2006-03-09, BUY CREATE, 3757.59
2006-03-10, BUY EXECUTED, Price: 3754.13, Cost: 3754.13, Comm 18.77
2006-04-11, SELL CREATE, 3788.81
2006-04-12, SELL EXECUTED, Price: 3786.93, Cost: 3754.13, Comm 18.93
2006-04-12, TRADE PROFIT, GROSS 32.80, NET -4.91
...
```

## 期货佣金（百分比且带保证金）

使用新参数，基于百分比的期货方案：

```shell
$ ./commission-schemes.py --commtype perc --percrel --comm 0.5 --margin 2000 --mult 10 --plot
```

不出所料，改变佣金后……最终结果也随之改变。

输出显示佣金现在是可变的：

```plaintext
2006-03-09, BUY CREATE, 3757.59
2006-03-10, BUY EXECUTED, Price: 3754.13, Cost: 2000.00, Comm 18.77
2006-04-11, SELL CREATE, 3788.81
2006-04-12, SELL EXECUTED, Price: 3786.93, Cost: 2000.00, Comm 18.93
2006-04-12, TRADE PROFIT, GROSS 328.00, NET 290.29
...
```

之前的运行设置了 2.0 货币单位（默认 `stake` 为 1）。

下一篇文章将详细说明新类和自定义佣金方案的实现。

## 示例

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse
import datetime

import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind


class SMACrossOver(bt.Strategy):
    params = (
        ('stake', 1),
        ('period', 30),
    )

    def log(self, txt, dt=None):
        ''' Logging function for this strategy'''
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # Buy/Sell order submitted/accepted to/by broker - Nothing to do
            return

        # Check if an order

 has been completed
        # Attention: broker could reject order if not enougth cash
        if order.status in [order.Completed, order.Canceled, order.Margin]:
            if order.isbuy():
                self.log(
                    'BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))
            else:  # Sell
                self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))

    def notify_trade(self, trade):
        if trade.isclosed:
            self.log('TRADE PROFIT, GROSS %.2f, NET %.2f' %
                     (trade.pnl, trade.pnlcomm))

    def __init__(self):
        sma = btind.SMA(self.data, period=self.p.period)
        # > 0 crossing up / < 0 crossing down
        self.buysell_sig = btind.CrossOver(self.data, sma)

    def next(self):
        if self.buysell_sig > 0:
            self.log('BUY CREATE, %.2f' % self.data.close[0])
            self.buy(size=self.p.stake)  # keep order ref to avoid 2nd orders

        elif self.position and self.buysell_sig < 0:
            self.log('SELL CREATE, %.2f' % self.data.close[0])
            self.sell(size=self.p.stake)


def runstrategy():
    args = parse_args()

    # Create a cerebro
    cerebro = bt.Cerebro()

    # Get the dates from the args
    fromdate = datetime.datetime.strptime(args.fromdate, '%Y-%m-%d')
    todate = datetime.datetime.strptime(args.todate, '%Y-%m-%d')

    # Create the 1st data
    data = btfeeds.BacktraderCSVData(
        dataname=args.data,
        fromdate=fromdate,
        todate=todate)

    # Add the 1st data to cerebro
    cerebro.adddata(data)

    # Add a strategy
    cerebro.addstrategy(SMACrossOver, period=args.period, stake=args.stake)

    # Add the commission - only stocks like a for each operation
    cerebro.broker.setcash(args.cash)

    commtypes = dict(
        none=None,
        perc=bt.CommInfoBase.COMM_PERC,
        fixed=bt.CommInfoBase.COMM_FIXED)

    # Add the commission - only stocks like a for each operation
    cerebro.broker.setcommission(commission=args.comm,
                                 mult=args.mult,
                                 margin=args.margin,
                                 percabs=not args.percrel,
                                 commtype=commtypes[args.commtype],
                                 stocklike=args.stocklike)

    # And run it
    cerebro.run()

    # Plot if requested
    if args.plot:
        cerebro.plot(numfigs=args.numfigs, volume=False)


def parse_args():
    parser = argparse.ArgumentParser(
        description='Commission schemes',
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,)

    parser.add_argument('--data', '-d',
                        default='../../datas/2006-day-001.txt',
                        help='data to add to the system')

    parser.add_argument('--fromdate', '-f',
                        default='2006-01-01',
                        help='Starting date in YYYY-MM-DD format')

    parser.add_argument('--todate', '-t',
                        default='2006-12-31',
                        help='Starting date in YYYY-MM-DD format')

    parser.add_argument('--stake', default=1, type=int,
                        help='Stake to apply in each operation')

    parser.add_argument('--period', default=30, type=int,
                        help='Period to apply to the Simple Moving Average')

    parser.add_argument('--cash', default=10000.0, type=float,
                        help='Starting Cash')

    parser.add_argument('--comm', default=2.0, type=float,
                        help=('Commission factor for operation, either a'
                              'percentage or a per stake unit absolute value'))

    parser.add_argument('--mult', default=10, type=int,
                        help='Multiplier for operations calculation')

    parser.add_argument('--margin', default=2000.0, type=float,
                        help='Margin for futures-like operations')

    parser.add_argument('--commtype', required=False, default='none',
                        choices=['none', 'perc', 'fixed'],
                        help=('Commission - choose none for the old'
                              ' CommissionInfo behavior'))

    parser.add_argument('--stocklike', required=False, action='store_true',
                        help=('If the operation is for stock-like assets or'
                              'future-like assets'))

    parser.add_argument('--percrel', required=False, action='store_true',
                        help=('If perc is expressed in relative xx% rather'
                              'than absolute value 0.xx'))

    parser.add_argument('--plot', '-p', action='store_true',
                        help='Plot the read data')

    parser.add_argument('--numfigs', '-n', default=1,
                        help='Plot using numfigs figures')

    return parser.parse_args()


if __name__ == '__main__':
    runstrategy()
```
