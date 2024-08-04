---
title: "创建/执行"
weight: 2
---

# 订单管理和执行

回测（backtesting），因此 backtrader，如果订单不能模拟，将是不完整的。为了实现这一点，平台中提供了以下内容。

对于订单管理，提供了3种基本操作：

- `buy`
- `sell`
- `cancel`

**注意：**
一个更新操作显然是合乎逻辑的，但常识告诉我们，这种方法主要用于使用判断性交易方法的手动操作员。

对于订单执行逻辑，提供以下执行类型：

- 市价订单（Market）
- 收盘价订单（Close）
- 限价订单（Limit）
- 止损订单（Stop）
- 止损限价订单（StopLimit）

### 订单管理

一些示例：

```python
# 购买主数据，使用sizer的默认股份，市价订单
order = self.buy()

# 市价订单 - 有效期将被 "忽略"
order = self.buy(valid=datetime.datetime.now() + datetime.timedelta(days=3))

# 市价订单 - 价格将被忽略
order = self.buy(price=self.data.close[0] * 1.02)

# 市价订单 - 手动股份
order = self.buy(size=25)

# 限价订单 - 想要设置价格并可以设置有效期
order = self.buy(exectype=Order.Limit,
                 price=self.data.close[0] * 1.02,
                 valid=datetime.datetime.now() + datetime.timedelta(days=3))

# 止损限价订单 - 想要设置价格，价格限制
order = self.buy(exectype=Order.StopLimit,
                 price=self.data.close[0] * 1.02,
                 plimit=self.data.close[0] * 1.07)

# 取消现有订单
self.broker.cancel(order)
```

**注意：**

所有订单类型都可以通过创建一个订单实例（或其子类之一）并传递给 broker 来创建：

```python
order = self.broker.submit(order)
```

**注意：**

Broker 中也有 `buy` 和 `sell` 操作，但它们对默认参数的容忍度较低。

### 订单执行逻辑

Broker 使用两个主要指导原则（假设？）进行订单执行。

1. 当前数据已经发生，不能用于执行订单。

    如果策略中的逻辑类似于：

    ```python
    if self.data.close > self.sma:  # 其中 sma 是简单移动平均线
        self.buy()
    ```

    期望不能是订单将以逻辑中检查的收盘价执行，因为它已经发生。

2. 订单可以在下一组开盘/最高/最低/收盘价格范围内首先执行（以及订单中设置的条件）。

3. 成交量不起作用

    实际交易中，如果交易者选择非流动性资产或恰好触及价格栏的高/低点，它实际上会起作用。

    但触及高/低点的情况很少发生（如果发生……你不需要 backtrader），所选择的资产将有足够的流动性来吸收任何正常交易的订单。

#### 市价订单（Market）
- 执行：
  - 下一个价格栏的开盘价（通常称为 bar）
- 理由：
  - 如果逻辑在时间点 X 执行并发出市价订单，下一个将发生的价格点是即将到来的开盘价。
- 注意：
  - 该订单始终执行，并忽略任何用于创建它的价格和有效期参数。

#### 收盘价订单（Close）
- 执行：
  - 使用下一个价格栏的收盘价，当下一个价格栏实际关闭时。
- 理由：
  - 大多数回测数据源已经包含已关闭的价格栏，订单将立即以下一个价格栏的收盘价执行。日数据源是最常见的示例。
  - 但系统可以被“tick”价格喂入，实际的价格栏（时间/日期）不断被新 tick 更新，而不会移动到下一个价格栏（因为时间和/或日期未改变）。
  - 只有当时间或日期改变时，价格栏才真正关闭，订单才执行。

#### 限价订单（Limit）
- 执行：
  - 如果数据触及在订单创建时设置的价格，则从下一个价格栏开始执行。
  - 如果设置了有效期并到达时间点，订单将被取消。
- 价格匹配：
  - backtrader 尝试为限价订单提供最现实的执行价格。
  - 使用4个价格点（开盘价/最高价/最低价/收盘价）可以部分推断出请求的价格是否可以改进。
  - 对于买入订单：
    - 情况1：如果价格栏的开盘价低于限价，订单立即以开盘价执行。订单在会话的开盘阶段被扫除。
    - 情况2：如果开盘价没有低于限价，但最低价低于限价，那么限价在会话期间已经出现，订单可以执行。
  - 对于卖出订单，逻辑显然是相反的。

#### 止损订单（Stop）
- 执行：
  - 如果数据触及在订单创建时设置的触发价格，则从下一个价格栏开始执行。
  - 如果设置了有效期并到达时间点，订单将被取消。
- 价格匹配：
  - backtrader 尝试为止损订单提供最现实的触发价格。
  - 使用4个价格点（开盘价/最高价/最低价/收盘价）可以部分推断出请求的价格是否可以改进。
  - 对于买入的止损订单：
    - 情况1：如果价格栏的开盘价高于止损价，订单立即以开盘价执行。旨在停止亏损，如果价格上涨对现有空头头寸不利。
    - 情况2：如果开盘价没有高于止损价，但最高价高于止损价，那么止损价在会话期间已经出现，订单可以执行。
  - 对于卖出订单，逻辑显然是相反的。

#### 止损限价订单（StopLimit）
- 执行：
  - 触发价格从下一个价格栏开始启动订单。
- 价格匹配：
  - 触发：使用止损匹配逻辑（但仅触发并将订单转变为限价订单）。
  - 限价：使用限价匹配逻辑。

### 一些示例
正如图片（带代码）所示，总是胜过几百万字的长篇解释。请注意，代码段集中在订单创建部分。完整代码在底部。

将使用一个价格在简单移动平均线上/下方收盘的策略来生成买入/卖出信号。

信号在图表底部可见：使用交叉指示器的交叉信号。

将保存生成的“买入”订单的引用，以只允许系统中最多同时存在一个订单。

#### 执行类型：市价订单（Market）
在图表中可以看到订单在生成信号后一根价格栏后以开盘价执行。

```python
if self.p.exectype == 'Market':
    self.buy(exectype=bt.Order.Market)  # 如果未给定，则默认

    self.log('BUY CREATE, exectype Market, price %.2f' %
             self.data.close[0])
```

#### 执行类型：收盘价订单（Close）
现在订单也是在生成信号后一根价格栏后执行，但以收盘价执行。

```python
elif self.p.exectype == 'Close':
    self.buy(exectype=bt.Order.Close)

    self.log('BUY CREATE, exectype Close, price %.2f' %
             self.data.close[0])
```

#### 执行类型：限价订单（Limit）
有效期在传递为参数时会提前计算。

```python
if self.p.valid:
    valid = self.data.datetime.date(0) +
            datetime.timedelta(days=self.p.valid)
else:
    valid = None
```

设置一个比信号生成价格（信号栏的收盘价）低1%的限价。请注意，这阻止了上面许多订单的执行。

```python
elif self.p.exectype == 'Limit':
    price = self.data.close * (1.0 - self.p.perc1 / 100.0)

    self.buy(exectype=bt.Order.Limit, price=price, valid=valid)

    if self.p.valid:
        txt = 'BUY CREATE, exectype Limit, price %.2f, valid: %s'
        self.log(txt % (price, valid.strftime('%Y-%m-%d')))
    else:
        txt = 'BUY CREATE, exectype Limit, price %.2f'
        self.log(txt % price)
```

#### 执行类型：带有效期的限价订单
为了不永远等待一个可能只在价格对“买入”订单不利时执行的限价订单，订单只会有效4个（日历）天。

```python
$ ./order-execution-samples.py --exectype Limit --perc1 1 --valid 4
```

#### 执行类型：止损订单（Stop）
设置一个比信号价格

高1%的止损价格。这意味着策略只有在信号生成后价格继续上涨时才买入，这可以解释为一种强劲信号。

```python
elif self.p.exectype == 'Stop':
    price = self.data.close * (1.0 + self.p.perc1 / 100.0)

    self.buy(exectype=bt.Order.Stop, price=price, valid=valid)

    if self.p.valid:
        txt = 'BUY CREATE, exectype Stop, price %.2f, valid: %s'
        self.log(txt % (price, valid.strftime('%Y-%m-%d')))
    else:
        txt = 'BUY CREATE, exectype Stop, price %.2f'
        self.log(txt % price)
```

#### 执行类型：止损限价订单（StopLimit）
设置一个比信号价格高1%的止损价格。但限价设置为比信号（收盘）价格高0.5%，这可以解释为：等待强劲信号的出现，但不要买入高点。等待下跌。

```python
elif self.p.exectype == 'StopLimit':
    price = self.data.close * (1.0 + self.p.perc1 / 100.0)

    plimit = self.data.close * (1.0 + self.p.perc2 / 100.0)

    self.buy(exectype=bt.Order.StopLimit, price=price, valid=valid,
             plimit=plimit)

    if self.p.valid:
        txt = ('BUY CREATE, exectype StopLimit, price %.2f,'
               ' valid: %s, pricelimit: %.2f')
        self.log(txt % (price, valid.strftime('%Y-%m-%d'), plimit))
    else:
        txt = ('BUY CREATE, exectype StopLimit, price %.2f,'
               ' pricelimit: %.2f')
        self.log(txt % (price, plimit))
```

### 测试脚本执行

详见命令行帮助：

```bash
$ ./order-execution-samples.py --help
usage: order-execution-samples.py [-h] [--infile INFILE]
                                  [--csvformat {bt,visualchart,sierrachart,yahoo,yahoo_unreversed}]
                                  [--fromdate FROMDATE] [--todate TODATE]
                                  [--plot] [--plotstyle {bar,line,candle}]
                                  [--numfigs NUMFIGS] [--smaperiod SMAPERIOD]
                                  [--exectype EXECTYPE] [--valid VALID]
                                  [--perc1 PERC1] [--perc2 PERC2]
```

完整代码：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse
import datetime
import os.path
import time
import sys

import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind

class OrderExecutionStrategy(bt.Strategy):
    params = (
        ('smaperiod', 15),
        ('exectype', 'Market'),
        ('perc1', 3),
        ('perc2', 1),
        ('valid', 4),
    )

    def log(self, txt, dt=None):
        ''' Logging function for this strategy'''
        dt = dt or self.data.datetime[0]
        if isinstance(dt, float):
            dt = bt.num2date(dt)
        print('%s, %s' % (dt.isoformat(), txt))

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # Buy/Sell order submitted/accepted to/by broker - Nothing to do
            self.log('ORDER ACCEPTED/SUBMITTED', dt=order.created.dt)
            self.order = order
            return

        if order.status in [order.Expired]:
            self.log('BUY EXPIRED')

        elif order.status in [order.Completed]:
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

        # Sentinel to None: new orders allowed
        self.order = None

    def __init__(self):
        # SimpleMovingAverage on main data
        # Equivalent to -> sma = btind.SMA(self.data, period=self.p.smaperiod)
        sma = btind.SMA(period=self.p.smaperiod)

        # CrossOver (1: up, -1: down) close / sma
        self.buysell = btind.CrossOver(self.data.close, sma, plot=True)

        # Sentinel to None: new orders allowed
        self.order = None

    def next(self):
        if self.order:
            # An order is pending ... nothing can be done
            return

        # Check if we are in the market
        if self.position:
            # In the market - check if it's the time to sell
            if self.buysell < 0:
                self.log('SELL CREATE, %.2f' % self.data.close[0])
                self.sell()

        elif self.buysell > 0:
            if self.p.valid:
                valid = self.data.datetime.date(0) + \
                        datetime.timedelta(days=self.p.valid)
            else:
                valid = None

            # Not in the market and signal to buy
            if self.p.exectype == 'Market':
                self.buy(exectype=bt.Order.Market)  # default if not given

                self.log('BUY CREATE, exectype Market, price %.2f' %
                         self.data.close[0])

            elif self.p.exectype == 'Close':
                self.buy(exectype=bt.Order.Close)

                self.log('BUY CREATE, exectype Close, price %.2f' %
                         self.data.close[0])

            elif self.p.exectype == 'Limit':
                price = self.data.close * (1.0 - self.p.perc1 / 100.0)

                self.buy(exectype=bt.Order.Limit, price=price, valid=valid)

                if self.p.valid:
                    txt = 'BUY CREATE, exectype Limit, price %.2f, valid: %s'
                    self.log(txt % (price, valid.strftime('%Y-%m-%d')))
                else:
                    txt = 'BUY CREATE, exectype Limit, price %.2f'
                    self.log(txt % price)

            elif self.p.exectype == 'Stop':
                price = self.data.close * (1.0 + self.p.perc1 / 100.0)

                self.buy(exectype=bt.Order.Stop, price=price, valid=valid)

                if self.p.valid:
                    txt = 'BUY CREATE, exectype Stop, price %.2f, valid: %s'
                    self.log(txt % (price, valid.strftime('%Y-%m-%d')))
                else:
                    txt = 'BUY CREATE, exectype Stop, price %.2f'
                    self.log(txt % price)

            elif self.p.exectype == 'StopLimit':
                price = self.data.close * (1.0 + self.p.perc1 / 100.0)

                plimit = self.data.close * (1.0 + self.p.perc2 / 100.0)

                self.buy(exectype=bt.Order.StopLimit, price=price, valid=valid,
                         plimit=plimit)

                if self.p.valid:
                    txt = ('BUY CREATE, exectype StopLimit, price %.2f,'
                           ' valid: %s, pricelimit: %.2f')
                    self.log(txt % (price, valid.strftime('%Y-%m-%d'), plimit))
                else:
                    txt = ('BUY CREATE, exectype StopLimit, price %.2f,'
                           ' pricelimit: %.2f')
                    self.log(txt % (price, plimit))

def runstrat():
    args = parse_args()

    cerebro = bt.Cerebro()

    data = getdata(args)
    cerebro.adddata(data)

    cerebro.addstrategy(
        OrderExecutionStrategy,
        exectype=args.exectype,
        perc1=args.perc1,
        perc2=args.perc2,
        valid=args.valid,
        smaperiod=args.smaperiod
    )
    cerebro.run()

    if args.plot:
        cerebro.plot(numfigs=args.numfigs, style=args.plotstyle)

def getdata(args):
    dataformat = dict(
        bt=btfeeds.BacktraderCSVData,
        visualchart=btfeeds.VChartCSVData,
        sierrachart=btfeeds.SierraChartCSVData,
        yahoo=btfeeds.YahooFinanceCSVData,
        yahoo_unreversed=btfeeds.YahooFinanceCSVData
    )

    dfkwargs = dict()
    if args.csvformat == 'yahoo_unreversed':
        dfkwargs['reverse'] = True

    if args.fromdate:
        fromdate = datetime.datetime.strptime(args.fromdate, '%Y-%m-%d')
        dfkwargs['fromdate'] = fromdate

    if args.todate:
        fromdate = datetime.datetime.strptime(args.todate, '%Y-%m-%d')
        dfkwargs['todate'] = todate

   

 dfkwargs['dataname'] = args.infile

    dfcls = dataformat[args.csvformat]

    return dfcls(**dfkwargs)

def parse_args():
    parser = argparse.ArgumentParser(
        description='Showcase for Order Execution Types')

    parser.add_argument('--infile', '-i', required=False,
                        default='../../datas/2006-day-001.txt',
                        help='File to be read in')

    parser.add_argument('--csvformat', '-c', required=False, default='bt',
                        choices=['bt', 'visualchart', 'sierrachart',
                                 'yahoo', 'yahoo_unreversed'],
                        help='CSV Format')

    parser.add_argument('--fromdate', '-f', required=False, default=None,
                        help='Starting date in YYYY-MM-DD format')

    parser.add_argument('--todate', '-t', required=False, default=None,
                        help='Ending date in YYYY-MM-DD format')

    parser.add_argument('--plot', '-p', action='store_false', required=False,
                        help='Plot the read data')

    parser.add_argument('--plotstyle', '-ps', required=False, default='bar',
                        choices=['bar', 'line', 'candle'],
                        help='Plot the read data')

    parser.add_argument('--numfigs', '-n', required=False, default=1,
                        help='Plot using n figures')

    parser.add_argument('--smaperiod', '-s', required=False, default=15,
                        help='Simple Moving Average Period')

    parser.add_argument('--exectype', '-e', required=False, default='Market',
                        help=('Execution Type: Market (default), Close, Limit,'
                              ' Stop, StopLimit'))

    parser.add_argument('--valid', '-v', required=False, default=0, type=int,
                        help='Validity for Limit sample: default 0 days')

    parser.add_argument('--perc1', '-p1', required=False, default=0.0,
                        type=float,
                        help=('%% distance from close price at order creation'
                              ' time for the limit/trigger price in Limit/Stop'
                              ' orders'))

    parser.add_argument('--perc2', '-p2', required=False, default=0.0,
                        type=float,
                        help=('%% distance from close price at order creation'
                              ' time for the limit price in StopLimit orders'))

    return parser.parse_args()

if __name__ == '__main__':
    runstrat()
```
