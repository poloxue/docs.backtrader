---
title: "交易监控"
weight: 7
---

# 交易监控

我们已经知道如何使用 backtrader 买卖交易了。本节将介绍 backtrader 如何监控它每笔交易，如成本、利润和佣金。由于佣金的存在，利润还分为毛利润和净利润

## 设置佣金

让我们先设置一个合理佣金率 - 0.1% （买入和卖出都要收取的），一行代码即可。

```python
# 0.1% ... 除以 100 以去掉百分号
cerebro.broker.setcommission(commission=0.001)
```

## 订单的成本和佣金

订单的成本和佣金可从订单回调 `notify_order` 中获取，它的 Order 参数 `order.executed.comm` 即为订单的已执行佣金，`order.executed.value` 即投入的成本。

```python
class TestStrategy(bt.Strategy):
    def notify_order(self, order):
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    '买入执行，价格：%.2f，成本：%.2f，佣金 %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))
            else:
                self.log('卖出执行，价格：%.2f，成本：%.2f，佣金 %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
```

## 交易记录计算利润

对于利润的计算，使用交易记录会更简单，所谓成交记录，即成交撮合一笔记录一次。与订单类似，我们可通过 `notify_trade` 获取成交记录。它的回调参数是 Trade 类对象，利润相关属性有 `trade.pnl`（毛利润）和 `trade.pnlcomm`（净利润）。

```python
class TestStrategy(bt.Strategy):
    def notify_trade(self, trade):
        if not trade.isclosed:
            return

        self.log('利润记录，毛利润 %.2f，净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))
```

因为只有是平仓交易，才有利润的说法，故通过 `trade.isclosed` 判断在平仓交易的情况才输出利润信息。

## 完整示例

```python
import datetime  
import os.path  
import sys

import backtrader as bt

class TestStrategy(bt.Strategy):

    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close

        self.order = None
        self.buyprice = None
        self.buycomm = None

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return

        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    '买入执行，价格：%.2f，成本：%.2f，佣金 %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:
                self.log('卖出执行，价格：%.2f，成本：%.2f，佣金 %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('订单取消/保证金不足/拒绝')

        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return

        self.log('利润记录，毛利润 %.2f，净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        self.log('收盘价，%.2f' % self.dataclose[0])
        if self.order:
            return

        if not self.position:
            if self.dataclose[0] < self.dataclose[-1]:
                if self.dataclose[-1] < self.dataclose[-2]:
                    self.log('创建买入订单，%.2f' % self.dataclose[0])
                      self.order = self.buy()
        else:
            if len(self) >= (self.bar_executed + 5):
                self.log('创建卖出订单，%.2f' % self.dataclose[0])
                self.order = self.sell()

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.addstrategy(TestStrategy)
    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    datapath = os.path.join(modpath, './datas/orcl-1995-2014.txt')

    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        fromdate=datetime.datetime(2000, 1, 1),
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    cerebro.adddata(data)
    cerebro.broker.setcash(100000.0)
    cerebro.broker.setcommission(commission=0.001)

    print('初始投资组合价值：%.2f' % cerebro.broker.getvalue())
    cerebro.run()
    print('最终投资组合价值：%.2f' % cerebro.broker.getvalue())
```

执行后的输出是：

```
初始投资组合价值：100000.00
2000-01-03T00:00:00, 收盘价，27.85
2000-01-04T00:00:00, 收盘价，25.39
2000-01-05T00:00:00, 收盘价，24.05
2000-01-05T00:00:00, 创建买入订单，24.05
2000-01-06T00:00:00, 买入执行，价格：23.61，成本：23.61，佣金 0.02
2000-01-06T00:00:00, 收盘价，22.63
2000-01-07T00:00:00, 收盘价，24.37
2000-01-10T00:00:00, 收盘价，27.29
2000-01-11T00:00:00, 收盘价，26.49
2000-01-12T00:00:00, 收盘价，24.90
2000-01-13T00:00:00, 收盘价，24.77
2000-01-13T00:00:00, 创建卖出订单，24.77
2000-01-14T00:00:00, 卖出执行，价格：25.70，成本：25.70，佣金 0.03
2000-01-14T00:00:00, 利润记录，毛利润 2.09，净利润 2.04
2000-01-14T00:00:00, 收盘价，25.18
...
...
...
2000-12-15T00:00:00, 创建卖出订单，26.93
2000-12-18T00:00:00, 卖出执行，价格：28.29，成本：28.29，佣金 0.03
2000-12-18T00:00:00, 利润记录，毛利润 -0.06，净利润 -0.12
2000-12-18T00:00:00, 收盘价，30.18
2000-12-19T00:00:00, 收盘价，28.88
2000-12-20T00:00:00, 收盘价，26.88
2000-12-20T00:00:00, 创建买入订单，26.88
2000-12-21T00:00:00, 买入执行，价格：26.23，成本：26.23，佣金 0.03
2000-12-21T00:00:00, 收盘价，27.82
2000-12-22T00:00:00, 收盘价，30.06
2000-12-26T00:00:00, 收盘价，29.17
2000-12-27T00:00:00, 收盘价，28.94
2000-12-28T00:00:00, 收盘价，29.29
2000-12-29T00:00:00, 收盘价，27.41
2000-12-29T00:00:00, 创建卖出订单，27.41
最终投资组合价值：100016.98
```

让我们将其中的 "利润记录" 的日志提取出来。

```
2000-01-14T00:00:00, 利润记录，毛利润 2.09，净利润 2.04
2000-02-07T00:00:00, 利润记录，毛利润 3.68，净利润 3.63
2000-02-28T00:00:00, 利润记录，毛利润 4.48，净利润 4.42
2000-03-13T00:00:00, 利润记录，毛利润 3.48，净利润 3.41
2000-03-22T00:00:00, 利润记录，毛利润 -0.41，净利润 -0.49
2000-04-07T00:00:00, 利润记录，毛利润 2.45，净利润 2.37
2000-04-20T00:00:00, 利润记录，毛利润 -1.95，净利润 -2.02
2000-05-02T00:00:00, 利润记录，毛利润 5.46，净利润 5.39
2000-05-11T00:00:00, 利润记录，毛利润 -3.74，净利润 -3.81
2000-05-30T00:00:00, 利润记录，毛利润 -1.46，净利润 -1.53
2000-07-05T00:00:00, 利润记录，毛利润 -1.62，净利润 -1.69
2000-07-14T00:00:00, 利润记录，毛利润 2.08，净利润 2.01
2000-07-28T00:00:00, 利润记录，毛利润 0.14，净利润 0.07
2000-08-08T00:00:00, 利润记录，毛利润 4.36，净利润 4.29
2000-08-21T00:00:00, 利润记录，毛利润 1.03，净利润 0.95
2000-09-15T00:00:00, 利润记录，毛利润 -4.26，净利润 -4.34
2000-09-27T00:00:00, 利润记录，毛利润 1.29，净利润 1.22
2000-10-13T00:00:00, 利润记录，毛利润 -2.98，净利润 -3.04
2000-10-26T00:00:00, 利润记录，毛利润 3.01，净利润 2.95
2000-11-06T00:00:00, 利润记录，毛利润 -3.59，净利润 -3.65
2000-11-16T00:00:00, 利润记录，毛利润 1.28，净利润 1.23
2000-12-01T00:00:00, 利润记录，毛利润 2.59，净利润 2.54
2000-12-18T00:00:00, 利润记录，毛利润 -0.06，净利润 -0.12
```

将这些 "净利润" 相加，最终的数字是 `15.83`。但系统告诉我们最终投资组合价值是 `100016.98`。显然，`15.83` 并不等于 `16.98`。

这其实并没有任何错误，净利润 `15.83` 是已平仓的交易利润。而最后一天实际还有一个未平仓头寸。虽然已发送了卖出操作，还要等待下个 bar 才能成交。最终投资组合价值止于 2000-12-29 的收盘价。这个订单的成交价格将会在下一个交易日（即 2001-01-02）被设定。

假设，我们将数据源提供到下一个交易日。

```
2001-01-02T00:00:00, 卖出执行，价格：27.87，成本：27.87，佣金 0.03
2001-01-02T00:00:00, 利润记录，毛利润 1.64，净利润 1.59
2001-01-02T00:00:00, 收盘价，24.87
2001-01-02T00:00:00, 创建买入订单，24.87
最终投资组合价值：100017.41
```

现在将先前的净利润与已完成操作的净利润相加：

```
15.83 + 1.59 = 17.42
```

忽略四舍五入的误差，现在一切就对上了。
