---
title: "技术指标"
weight: 9
---

# 技术指标（Indicators）

本节将介绍如何使用技术指标，将技术指标作为入场和出场信号。我们将用简单移动平均线（Simple Moving Average），或称 SMA，作为演示指标。SMA 是一个非常简单的技术指标，计算一定周期的价格均值。

## 交易规则

基于 SMA 交易规则，定义如下所示：

- **入场条件：** 当收盘价大于最新的 SMA，则入场买入。  
- **出场条件：** 当持有头寸，当收盘价小于 SMA，则出场卖出。

前面章节的策略代码大部分可复用，现在重点关注如何计算技术指标。

## 指标计算

backtrader 下的 `indicators` 模块内置了大量技术指标的计算方法，如 SMA 简单移动均线的计算。

```python
self.sma = bt.indicators.MovingAverageSimple(self.datas[0], period=self.params.maperiod)
```

如上代码中参数 `self.params.maperiod` 就是 SMA 的均线周期。

注：如果安装了 talib，backtrader 也集成了 talib 的支持，详情文档 [指标-TALib](/backtrader/docs/08-indicators/04-talib/)。

## 条件判断

现在基于 `self.sma` 判断进出场条件。

为了简化代码，这里只考虑 SMA 的判断逻辑，在完整实例中会包含所有情况。

入场判断：

```python
self.dataclose[0] > self.sma[0]
```

出场判断：

```python
self.dataclose[0] < self.sma[0]
```

## 策略代码

起始现金 1000 货币单位。

```python
import datetime  # For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt
```

策略部分：

```python
class TestStrategy(bt.Strategy):
    params = (
        ('maperiod', 15),
    )

    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.buyprice = None
        self.buycomm = None
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod)

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return

        if order.status in [order.Completed]:
            if order.isbuy():
                self.log('BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:
                self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order Canceled/Margin/Rejected')

        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('OPERATION PROFIT, GROSS %.2f, NET %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        self.log('Close, %.2f' % self.dataclose[0])

        if self.order:
            return

        if not self.position:
            if self.dataclose[0] > self.sma[0]:
                self.log('BUY CREATE, %.2f' % self.dataclose[0])
                self.order = self.buy()
        else:
            if self.dataclose[0] < self.sma[0]:
                self.log('SELL CREATE, %.2f' % self.dataclose[0])
                self.order = self.sell()

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.addstrategy(TestStrategy)

    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    datapath = os.path.join(modpath, '../../datas/orcl-1995-2014.txt')

    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        fromdate=datetime.datetime(2000, 1, 1),
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    cerebro.adddata(data)
    cerebro.broker.setcash(1000.0)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    cerebro.broker.setcommission(commission=0.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

输出：

```
Starting Portfolio Value: 1000.00
2000-01-24, Close, 25.55
2000-01-25, Close, 26.61
2000-01-25, BUY CREATE, 26.61
2000-01-26, BUY EXECUTED, Size 10, Price: 26.76, Cost: 267.60, Commission 0.00
2000-01-26, Close, 25.96
2000-01-27, Close, 24.43
2000-01-27, SELL CREATE, 24.43
2000-01-28, SELL EXECUTED, Size 10, Price: 24.28, Cost: 242.80, Commission 0.00
2000-01-28, OPERATION PROFIT, GROSS -24.80, NET -24.80
...
...
...
2000-12-20, SELL CREATE, 26.88
2000-12-21, SELL EXECUTED, Size 10, Price: 26.23, Cost: 262.30, Commission 0.00
2000-12-21, OPERATION PROFIT, GROSS -20.60, NET -20.60
2000-12-21, Close, 27.82
2000-12-21, BUY CREATE, 27.82
2000-12-22, BUY EXECUTED, Size 10, Price: 28.65, Cost: 286.50, Commission 0.00
2000-12-22, Close, 30.06
2000-12-26, Close,

 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
2000-12-29, SELL CREATE, 27.41
Final Portfolio Value: 973.90
```

现在，投资组合变得亏损了。

