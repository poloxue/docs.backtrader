---
title: "Bracket Orders"
weight: 5
---

# Bracket Orders

1.9.37.116版本增加了Bracket订单，为回测经纪商提供了广泛的订单支持（Market、Limit、Close、Stop、StopLimit、StopTrail、StopTrailLimit、OCO）。

**注意**，这是为回测和 Interactive Brokers 实现的。

Bracket订单不是单个订单，而是由3个订单组成。

以做多为例：

1. 一个主要的买单，通常设置为 Limit 或 StopLimit 订单。
2. 一个低价卖单，通常设置为 Stop 订单以限制损失。
3. 一个高价卖单，通常设置为 Limit 订单以获取利润。

做空也有对应的卖单和 2 个买单。

低价/高价卖单实际上形成了一个围绕主要订单的 Bracket。

为了使其合理，以下规则适用：

- 3个订单一起提交，以避免其中任何一个独立触发。
- 低价/高价卖单被标记为主要订单的子订单。
- 子订单在主要订单执行之前不活跃。
- 取消主要订单会取消低价和高价卖单。
- 执行主要订单会激活低价和高价卖单。
- 一旦活跃，低价/高价卖单的执行或取消会自动取消另一个订单。

## 使用模式

有两种方式创建 Bracket 订单组：

1. 单次发布3个订单。
2. 手动发布3个订单。

### 单次发布Bracket

backtrader在Strategy中提供了两个新方法来控制Bracket订单：`buy_bracket`和`sell_bracket`。

**注意**

签名和信息见下文或Strategy参考部分。

通过单个语句完成3个订单的设置。示例如下：

```python
brackets = self.buy_bracket(limitprice=14.00, price=13.50, stopprice=13.00)
```

注意，`stopprice`和`limitprice`围绕`price`设定。这应该足够了。

实际的目标数据将是 data0，大小将由默认的sizer自动确定。当然，可以指定其他参数来精细控制执行。

返回值是一个包含3个订单的列表：[主要订单，stop订单，limit订单]。

因为在发布 `sell_bracket` 订单时，低价和高价将翻转，所以参数命名遵循约定：`stop` 用于止损（在做多操作中是低价，在做空操作中是高价），`limit` 用于获取利润（在做多操作中是高价，在做空操作中是低价）。

### 手动发布Bracket

这涉及生成3个订单，并处理`transmit`和`parent`参数。规则如下：

1. 必须首先创建主要订单并设置`transmit=False`。
2. 低价/高价订单必须有`parent=main_side_order`。
3. 第一个创建的低价/高价订单必须设置`transmit=False`。
4. 最后一个创建的订单（无论是低价还是高价）设置`transmit=True`。

以下示例实现了与上述单次命令相同的效果：

```python
mainside = self.buy(price=13.50, exectype=bt.Order.Limit, transmit=False)
lowside  = self.sell(price=13.00, size=mainside.size, exectype=bt.Order.Stop, transmit=False, parent=mainside)
highside = self.sell(price=14.00, size=mainside.size, exectype=bt.Order.Limit, transmit=True, parent=mainside)
```

需要做更多的事情：

- 跟踪主要订单，指示它是其他订单的父订单。
- 控制`transmit`以确保只有最后一个订单触发联合传输。
- 指定执行类型。
- 为低价和高价订单指定大小。

因为大小必须相同。如果未手动指定大小且用户引入了sizer，sizer可能会为订单指示不同的值。因此，需要在调用时手动添加大小。

## 示例

运行下面的示例生成如下输出（为简洁起见进行了截断）：

```bash
$ ./bracket.py --plot

2005-01-28: Oref 1 / Buy at 2941.11055
2005-01-28: Oref 2 / Sell Stop at 2881.99275
2005-01-28: Oref 3 / Sell Limit at 3000.22835
2005-01-31: Order ref: 1 / Type Buy / Status Submitted
2005-01-31: Order ref: 2 / Type Sell / Status Submitted
2005-01-31: Order ref: 3 / Type Sell / Status Submitted
2005-01-31: Order ref: 1 / Type Buy / Status Accepted
2005-01-31: Order ref: 2 / Type Sell / Status Accepted
2005-01-31: Order ref: 3 / Type Sell / Status Accepted
2005-02-01: Order ref: 1 / Type Buy / Status Expired
2005-02-01: Order ref: 2 / Type Sell / Status Canceled
2005-02-01: Order ref: 3 / Type Sell / Status Canceled
...
2005-08-11: Oref 16 / Buy at 3337.3892
2005-08-11: Oref 17 / Sell Stop at 3270.306
2005-08-11: Oref 18 / Sell Limit at 3404.4724
2005-08-12: Order ref: 16 / Type Buy / Status Submitted
2005-08-12: Order ref: 17 / Type Sell / Status Submitted
2005-08-12: Order ref: 18 / Type Sell / Status Submitted
2005-08-12: Order ref: 16 / Type Buy / Status Accepted
2005-08-12: Order ref: 17 / Type Sell / Status Accepted
2005-08-12: Order ref: 18 / Type Sell / Status Accepted
2005-08-12: Order ref: 16 / Type Buy / Status Completed
2005-08-18: Order ref: 17 / Type Sell / Status Completed
2005-08-18: Order ref: 18 / Type Sell / Status Canceled
...
2005-09-26: Oref 22 / Buy at 3383.92535
2005-09-26: Oref 23 / Sell Stop at 3315.90675
2005-09-26: Oref 24 / Sell Limit at 3451.94395
2005-09-27: Order ref: 22 / Type Buy / Status Submitted
2005-09-27: Order ref: 23 / Type Sell / Status Submitted
2005-09-27: Order ref: 24 / Type Sell / Status Submitted
2005-09-27: Order ref: 22 / Type Buy / Status Accepted
2005-09-27: Order ref: 23 / Type Sell / Status Accepted
2005-09-27: Order ref: 24 / Type Sell / Status Accepted
2005-09-27: Order ref: 22 / Type Buy / Status Completed
2005-10-04: Order ref: 24 / Type Sell / Status Completed
2005-10-04: Order ref: 23 / Type Sell / Status Canceled
...
```

显示了 3 种不同的结果：

1. 第一种情况下，主要订单过期，自动取消了其他两个订单。
2. 第二种情况下，主要订单完成，低价订单（买入情况下的止损）执行，限制了损失。
3. 第三种情况下，主要订单完成，高价订单（限价）执行。

可以注意到，已完成的订单id是22和24，高价订单最后发布，未执行的低价订单id是23。

**图示**

可以立即看到，亏损交易集中在相同的值附近，而盈利交易也是如此，这是 Bracket 的目的。控制两侧。

运行的示例手动发布3个订单，但可以使用`buy_bracket`。输出如下：

```bash
$ ./bracket.py --strat usebracket=True
```

结果相同。

## 参考

请参阅新的`buy_bracket`和`sell_bracket`方法

```python
def buy_bracket(self, data=None, size=None, price=None, plimit=None,
                exectype=bt.Order.Limit, valid=None, tradeid=0,
                trailamount=None, trailpercent=None, oargs={},
                stopprice=None, stopexec=bt.Order.Stop, stopargs={},
                limitprice=None, limitexec=bt.Order.Limit, limitargs={},
                **kwargs):
    '''
    创建一个Bracket订单组（低侧 - 买单 - 高侧）。默认行为如下：

      - 发出执行类型为“Limit”的**买单**

      - 发出执行类型为“Stop”的*低侧*Bracket**卖单**

      - 发出执行类型为“Limit”的*高侧*Bracket**卖单**。

    参见下文以了解不同参数的含义

      - ``data``（默认：``None``）

        订单针对的数据。如果为``None``，则使用系统中的第一个数据，即``self.datas[0

]或self.data0``（即``self.data``）

      - ``size``（默认：``None``）

        订单的数据单位大小（正数）。

        如果为``None``，则使用通过``getsizer``检索到的``sizer``实例来确定大小。

        **注意**：相同的大小适用于Bracket的所有3个订单

      - ``price``（默认：``None``）

        使用的价格（实时经纪商可能会对实际格式施加限制，如果不符合最小价格单位要求）

        ``None``对于``Market``和``Close``订单是有效的（市场决定价格）

        对于``Limit``、``Stop``和``StopLimit``订单，此值确定触发点（在``Limit``的情况下，触发点显然是订单应匹配的价格）

      - ``plimit``（默认：``None``）

        仅适用于``StopLimit``订单。这是设置隐含限价订单的价格，一旦触发了``Stop``（使用``price``）

      - ``trailamount``（默认：``None``）

        如果订单类型为StopTrail或StopTrailLimit，这是一个绝对金额，确定价格的距离（卖单下方和买单上方），以保持追踪止损

      - ``trailpercent``（默认：``None``）

        如果订单类型为StopTrail或StopTrailLimit，这是一个百分比金额，确定价格的距离（卖单下方和买单上方），以保持追踪止损（如果也指定了``trailamount``，将使用它）

      - ``exectype``（默认：``bt.Order.Limit``）

        可能的值：（参见``buy``方法的文档）

      - ``valid``（默认：``None``）

        可能的值：（参见``buy``方法的文档）

      - ``tradeid``（默认：``0``）

        可能的值：（参见``buy``方法的文档）

      - ``oargs``（默认：``{}``）

        要传递给主要订单的特定关键字参数（在``dict``中）。默认``**kwargs``中的参数将应用于此。

      - ``**kwargs``：其他经纪商实现可能支持的额外参数。``backtrader``将*kwargs*传递给创建的订单对象

        可能的值：（参见``buy``方法的文档）

        **注意**：此``kwargs``将应用于Bracket的所有3个订单。参见下文以了解低侧和高侧订单的特定关键字参数

      - ``stopprice``（默认：``None``）

        *低侧*止损订单的特定价格

      - ``stopexec``（默认：``bt.Order.Stop``）

        *低侧*订单的特定执行类型

      - ``stopargs``（默认：``{}``）

        要传递给低侧订单的特定关键字参数（在``dict``中）。默认``**kwargs``中的参数将应用于此。

      - ``limitprice``（默认：``None``）

        *高侧*止损订单的特定价格

      - ``stopexec``（默认：``bt.Order.Limit``）

        *高侧*订单的特定执行类型

      - ``limitargs``（默认：``{}``）

        要传递给高侧订单的特定关键字参数（在``dict``中）。默认``**kwargs``中的参数将应用于此。

    返回：
      - 一个包含3个Bracket订单的列表[订单，低侧，高侧]
    '''

def sell_bracket(self, data=None,
                 size=None, price=None, plimit=None,
                 exectype=bt.Order.Limit, valid=None, tradeid=0,
                 trailamount=None, trailpercent=None,
                 oargs={},
                 stopprice=None, stopexec=bt.Order.Stop, stopargs={},
                 limitprice=None, limitexec=bt.Order.Limit, limitargs={},
                 **kwargs):
    '''
    创建一个Bracket订单组（低侧 - 卖单 - 高侧）。默认行为如下：

      - 发出执行类型为“Limit”的**卖单**

      - 发出执行类型为“Stop”的*高侧*Bracket**买单**

      - 发出执行类型为“Limit”的*低侧*Bracket**买单**。

    参见``bracket_buy``以了解参数的含义

    返回：
      - 一个包含3个Bracket订单的列表[订单，低侧，限价侧]
    '''
```

## 示例用法

```bash
$ ./bracket.py --help
usage: bracket.py [-h] [--data0 DATA0] [--fromdate FROMDATE] [--todate TODATE]
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

## 示例代码

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)

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
        usebracket=False,  # use order_target_size
        switchp1p2=False,  # switch prices of order1 and order2
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

        if self.p.usebracket:
            print('-' * 5, 'Using buy_bracket')

    def next(self):
        if self.orefs:
            return  # pending orders do nothing

        if not self.position:
            if self.cross > 0.0:  # crossing up
                close = self.data.close[0]
                p1 = close * (1.0 - self.p.limit)
                p2 = p1 - 0.02 * close
                p3 = p1 + 0.02 * close

                valid1 = datetime.timedelta(self.p.limdays)
                valid2 = valid3 = datetime.timedelta(self.p.limdays2)

                if self.p.switchp1p2:
                    p1, p2 = p2, p1
                    valid1, valid2 = valid2, valid1

                if not self.p.usebracket:
                    o1 = self.buy(exectype=bt.Order.Limit, price=p1, valid=valid1, transmit=False)
                    print('{}: Oref {} / Buy at {}'.format(self.datetime.date(), o1.ref, p1))

                    o2 = self.sell(exectype=bt.Order.Stop, price=p2, valid=valid2, parent=o1, transmit=False)
                    print('{}: Oref {} / Sell Stop at {}'.format(self.datetime.date(), o2.ref, p2))

                    o3 = self.sell(exectype=bt.Order.Limit, price=p3, valid=valid3, parent=o1, transmit=True)
                    print('{}: Oref {} / Sell Limit at {}'.format(self.datetime.date(), o3.ref, p3))

                    self.orefs = [o1.ref, o2.ref, o3.ref]

                else:
                    os = self.buy_bracket(price=p1, valid=valid1, stopprice=p2, stopargs=dict(valid=valid2),
                                          limitprice=p3, limitargs=dict(valid=valid3),)
                    self.orefs = [o.ref for o in os]

        else:  # in the market
            if (len(self) - self.holdstart) >= self.p.hold:
                pass  # do nothing in this case

def runstrat(args=None):
    args = parse_args(args)
    cerebro = bt.Cerebro()

    # Data feed kwargs
    kwargs = dict()
    dtfmt, tmfmt

 = '%Y-%m-%d', 'T%H:%M:%S'
    for a, d in ((getattr(args, x), x) for x in ['fromdate', 'todate']):
        if a:
            strpfmt = dtfmt + tmfmt * ('T' in a)
            kwargs[d] = datetime.datetime.strptime(a, strpfmt)

    data0 = bt.feeds.BacktraderCSVData(dataname=args.data0, **kwargs)
    cerebro.adddata(data0)
    cerebro.broker = bt.brokers.BackBroker(**eval('dict(' + args.broker + ')'))
    cerebro.addsizer(bt.sizers.FixedSize, **eval('dict(' + args.sizer + ')'))
    cerebro.addstrategy(St, **eval('dict(' + args.strat + ')'))
    cerebro.run()

    if args.plot:
        cerebro.plot(**eval('dict(' + args.plot + ')'))

def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description='Sample Skeleton'
    )

    parser.add_argument('--data0', default='../../datas/2005-2006-day-001.txt',
                        required=False, help='Data to read in')
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
