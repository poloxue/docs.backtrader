---
title: "佣金"
weight: 1
---

# 佣金

## 中立性

在开始之前，让我们记住 backtrader 尝试保持数据代表内容的中立性。不同的佣金方案可以应用于相同的数据集。让我们看看如何做到这一点。

## 经纪商快捷方式

这使得最终用户远离 CommissionInfo 对象，因为可以通过一次函数调用创建/设置佣金方案。在常规的 cerebro 创建/设置过程中，只需在经纪商成员属性上添加一个调用 setcommission 的调用即可。

以下调用设置了使用 Interactive Brokers 操作 Eurostoxx50 期货的常规佣金方案：

```python
cerebro.broker.setcommission(commission=2.0, margin=2000.0, mult=10.0)
```

由于大多数用户通常只测试单一工具，这已经足够。

如果你为你的数据馈送命名，因为在图表上同时考虑了多个工具，这个调用可以稍微扩展如下：

```python
cerebro.broker.setcommission(commission=2.0, margin=2000.0, mult=10.0, name='Eurostoxxx50')
```

在这种情况下，此即时佣金方案将仅应用于名称匹配 Eurostoxx50 的工具。

## setcommission 参数的含义

- `commission`（默认值：0.0）

  每次操作的货币单位，绝对值或百分比。在上述示例中，每份合约的买入和卖出费用分别为 2.0 欧元。
  重要的是何时使用绝对值或百分比值。

  如果 margin 为 False（例如，它是 False、0 或 None），则将视为佣金表示为价格乘以操作数量的百分比。

  如果 margin 是其他值，则视为操作发生在类似期货的工具上，佣金是每张合约的固定价格。

- `margin`（默认值：None）

  操作期货类工具时需要的保证金。如上所述：

  如果没有设置 margin，则佣金将被视为百分比，并应用于买卖操作的价格 * 数量。

  如果设置了 margin，则佣金将被视为固定值，并乘以买卖操作的数量。

- `mult`（默认值：1.0）

  对于期货类工具，这决定了应用于损益计算的乘数。这使得期货同时具有吸引力和风险。

- `name`（默认值：None）

  将佣金方案应用于名称匹配的工具。可以在创建数据馈送时设置此值。如果未设置，则方案将适用于系统中的任何数据。

## 两个示例：股票 vs 期货

期货的示例：

```python
cerebro.broker.setcommission(commission=2.0, margin=2000.0, mult=10.0)
```

股票的示例：

```python
cerebro.broker.setcommission(commission=0.005)  # 交易金额的 0.5%
```

注意

第二种语法不设置 margin 和 mult，backtrader 试图通过将佣金视为百分比来进行智能处理。

要完全指定佣金方案，需要创建 CommissionInfo 的子类。

## 创建永久性佣金方案

可以通过直接使用 CommissionInfo 类创建更永久的佣金方案。用户可以选择在某处定义：

```python
import backtrader as bt

commEurostoxx50 = bt.CommissionInfo(commission=2.0, margin=2000.0, mult=10.0)
```

然后在另一个 Python 模块中应用：

```python
from mycomm import commEurostoxx50

...

cerebro.broker.addcommissioninfo(commEuroStoxx50, name='Eurostoxxx50')
```

CommissionInfo 是一个对象，使用与 backtrader 环境中的其他对象类似的 params 声明。因此上述内容也可以表示为：

```python
import backtrader as bt

class CommEurostoxx50(bt.CommissionInfo):
    params = dict(commission=2.0, margin=2000.0, mult=10.0)
```

然后：

```python
from mycomm import CommEurostoxx50

...

cerebro.broker.addcommissioninfo(CommEuroStoxx50(), name='Eurostoxxx50')
```

## 使用 SMA 交叉的实际比较

使用简单移动平均交叉作为进出信号，使用相同的数据集来测试期货类佣金方案和股票类佣金方案。

注意

期货头寸不仅可以给出进出行为，还可以在每次机会时进行反转行为。但这个示例是关于比较佣金方案的。

代码（见底部完整策略）相同，可以在定义策略之前选择方案。

```python
futures_like = True

if futures_like:
    commission, margin, mult = 2.0, 2000.0, 10.0
else:
    commission, margin, mult = 0.005, None, 1
```

只需将 futures_like 设置为 false 即可使用股票类方案运行。

一些日志代码已被添加以评估不同佣金方案的影响。让我们集中在前两个操作。

对于期货：

```plaintext
2006-03-09, BUY CREATE, 3757.59
2006-03-10, BUY EXECUTED, Price: 3754.13, Cost: 2000.00, Comm 2.00
2006-04-11, SELL CREATE, 3788.81
2006-04-12, SELL EXECUTED, Price: 3786.93, Cost: 2000.00, Comm 2.00
2006-04-12, OPERATION PROFIT, GROSS 328.00, NET 324.00
2006-04-20, BUY CREATE, 3860.00
2006-04-21, BUY EXECUTED, Price: 3863.57, Cost: 2000.00, Comm 2.00
2006-04-28, SELL CREATE, 3839.90
2006-05-02, SELL EXECUTED, Price: 3839.24, Cost: 2000.00, Comm 2.00
2006-05-02, OPERATION PROFIT, GROSS -243.30, NET -247.30
```

对于股票：

```plaintext
2006-03-09, BUY CREATE, 3757.59
2006-03-10, BUY EXECUTED, Price: 3754.13, Cost: 3754.13, Comm 18.77
2006-04-11, SELL CREATE, 3788.81
2006-04-12, SELL EXECUTED, Price: 3786.93, Cost: 3786.93, Comm 18.93
2006-04-12, OPERATION PROFIT, GROSS 32.80, NET -4.91
2006-04-20, BUY CREATE, 3860.00
2006-04-21, BUY EXECUTED, Price: 3863.57, Cost: 3863.57, Comm 19.32
2006-04-28, SELL CREATE, 3839.90
2006-05-02, SELL EXECUTED, Price: 3839.24, Cost: 3839.24, Comm 19.20
2006-05-02, OPERATION PROFIT, GROSS -24.33, NET -62.84
```

第一个操作的价格如下：

- 买入（执行）：3754.13
- 卖出（执行）：3786.93

期货的损益（含佣金）：324.0

股票的损益（含佣金）：-4.91

佣金完全吞噬了股票操作的利润，而期货操作的利润只受到轻微影响。

第二个操作：

- 买入（执行）：3863.57
- 卖出（执行）：3389.24

期货的损益（含佣金）：-247.30

股票的损益（含佣金）：-62.84

对于此负操作，期货的影响明显更大。

但：

期货累计净利润和损失：324.00 + (-247.30) = 76.70

股票累计净利润和损失：(-4.91) + (-62.84) = -67.75

在下图中可以看到累计效果，还可以看到在全年结束时，期货产生了更大的利润，但也遭受了更大的回撤（更深的水下）

但重要的是：无论是期货还是股票……都可以进行回测。

## 代码

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind

futures_like = True

if futures_like:
    commission, margin, mult = 2.0, 2000.0, 10.0
else:
    commission, margin, mult = 0.

005, None, 1


class SMACrossOver(bt.Strategy):
    def log(self, txt, dt=None):
        ''' Logging function for this strategy'''
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def notify(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # Buy/Sell order submitted/accepted to/by broker - Nothing to do
            return

        # Check if an order has been completed
        # Attention: broker could reject order if not enough cash
        if order.status in [order.Completed, order.Canceled, order.Margin]:
            if order.isbuy():
                self.log(
                    'BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
                self.opsize = order.executed.size
            else:  # Sell
                self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))

                gross_pnl = (order.executed.price - self.buyprice) * \
                    self.opsize

                if margin:
                    gross_pnl *= mult

                net_pnl = gross_pnl - self.buycomm - order.executed.comm
                self.log('OPERATION PROFIT, GROSS %.2f, NET %.2f' %
                         (gross_pnl, net_pnl))

    def __init__(self):
        sma = btind.SMA(self.data)
        # > 0 crossing up / < 0 crossing down
        self.buysell_sig = btind.CrossOver(self.data, sma)

    def next(self):
        if self.buysell_sig > 0:
            self.log('BUY CREATE, %.2f' % self.data.close[0])
            self.buy()  # keep order ref to avoid 2nd orders

        elif self.position and self.buysell_sig < 0:
            self.log('SELL CREATE, %.2f' % self.data.close[0])
            self.sell()


if __name__ == '__main__':
    # Create a cerebro entity
    cerebro = bt.Cerebro()

    # Add a strategy
    cerebro.addstrategy(SMACrossOver)

    # Create a Data Feed
    datapath = ('../../datas/2006-day-001.txt')
    data = bt.feeds.BacktraderCSVData(dataname=datapath)

    # Add the Data Feed to Cerebro
    cerebro.adddata(data)

    # set commission scheme -- CHANGE HERE TO PLAY
    cerebro.broker.setcommission(
        commission=commission, margin=margin, mult=mult)

    # Run over everything
    cerebro.run()

    # Plot the result
    cerebro.plot()
```

## 参考

```python
class backtrader.CommInfoBase()
```

基础类用于佣金方案。

参数：

- `commission`（默认值：0.0）：基础佣金值，百分比或货币单位
- `mult`（默认值：1.0）：应用于资产价值/利润的乘数
- `margin`（默认值：None）：开/持仓所需的货币单位，仅当类中的 `_stocklike` 属性设置为 False 时适用
- `automargin`（默认值：False）：用于方法 `get_margin`，自动计算所需的保证金/保证金
- `commtype`（默认值：None）：支持的值为 CommInfoBase.COMM_PERC（佣金理解为 %）和 CommInfoBase.COMM_FIXED（佣金理解为货币单位）

默认值 None 是一个支持的值，用于保留与旧的 CommissionInfo 对象的兼容性。如果 commtype 设置为 None，则以下适用：

margin 为 None：内部 `_commtype` 设置为 COMM_PERC，`_stocklike` 设置为 True（按百分比操作股票）

margin 不是 None：`_commtype` 设置为 COMM_FIXED，`_stocklike` 设置为 False（按固定回合佣金操作期货）

如果此参数设置为其他值，则将传递给内部 `_commtype` 属性，并且相同将适用于参数 stocklike 和内部属性 `_stocklike`

- `stocklike`（默认值：False）： 指示工具是类似股票还是类似期货（请参阅上述 commtype 讨论）
- `percabs`（默认值：False）： 当 commtype 设置为 COMM_PERC 时，参数 commission 是否理解为 XX% 或 0.XX

  如果此参数为 True：0.XX

  如果此参数为 False：XX%

- `interest`（默认值：0.0）：如果这是非零值，则这是持有空头头寸的年利率。这主要是指股票空头卖空

  公式：天数 * 价格 * 绝对值（大小）*（利息 / 365）

  必须以绝对值表示：0.05 -> 5%

  注意，可以通过覆盖方法 `_get_credit_interest` 更改行为

- `interest_long`（默认值：False）：某些产品如 ETF 对空头和多头头寸收取利息。如果为 True 并且 interest 为非零，则对两个方向都收取利息

- `leverage`（默认值：1.0）：与所需现金相比的杠杆比例

## CommissionInfo 类

基础类用于实际佣金方案。CommInfoBase 是为了保持对 backtrader 提供的原始不完整支持。新的佣金方案从此类派生。默认的 percabs 值也更改为 True .

参数：

- `percabs`（默认值：True）：当 commtype 设置为 COMM_PERC 时，参数 commission 是否理解为 XX% 或 0.XX

  如果此参数为 True：0.XX

  如果此参数为 False：XX%

返回此佣金方案允许的杠杆水平

```python
get_leverage()
```


返回在给定价格下满足现金操作所需的大小
```python
getsize(price, cash)
```

返回操作所需的现金量

```python
getoperationcost(size, price)
```

返回给定价格的大小值。对于类似期货的对象，固定为大小 * 保证金

```python
getvaluesize(size, price)
```

返回给定价格的头寸价值。对于类似期货的对象，固定为大小 * 保证金

```python
getvalue(position, price)
```


返回在给定价格下单个资产所需的实际保证金/保证金。默认实现有以下策略：

使用参数 margin 如果参数 automargin 评估为 False

使用参数 mult，即 mult * price 如果 automargin < 0

使用参数 automargin，即 automargin * price 如果 automargin > 0

```python
get_margin(price)
```

计算在给定价格下的操作佣金

```python
getcommission(size, price)
```

计算在给定价格下的操作佣金

pseudoexec：如果为 True，则操作尚未执行

```python
_getcommission(size, price, pseudoexec)
```

返回头寸的实际损益

```python
profitandloss(size, price, newprice)
```

计算价格差异的现金调整

```python
cashadjust(size, price, newprice)
```

计算股票卖空或特定产品的信用费用

```python
get_credit_interest(data, pos, dt)
```

此方法返回由经纪商收取的信用利息费用。

在 size > 0 的情况下，仅当类的参数 interest_long 为 True 时才调用此方法

计算信用利率的公式是：公式：天数 * 价格 * 绝对值（大小）*（利息 / 365）

```python
_get_credit_interest(data, size, price, days, dt0, dt1)
```

参数：

- `data`：收取利息的数据馈送
- `size`：当前头寸大小。> 0 表示多头头寸，< 0 表示空头头寸（此参数不会为 0）
- `price`：当前头寸价格
- `days`：自上次信用计算以来经过的天数（这是（dt0 - dt1）.days）
- `dt0`：（datetime.datetime）当前日期时间
- `dt1`：（datetime.datetime）上次计算的日期时间

默认实现中不使用 dt0 和 dt1，它们作为覆盖方法的额外输入提供。
