---
title: "可视化"
weight: 11
---

# 可视化

通过 `print` 输出每个 bar 的信息不利于我们阅读，我们还是更倾向于图表的视觉效果。backtrader 内置了图表绘制的能力，一行代码即可绘图。

```python
cerebro.plot()
```
请确保在调用`cerebro.run()`之后执行，还有，`backtrader` 的绘图能力依赖 matplotlib。

## 演示

为了展示出基本的价格和收益外，我们将执行以下操作以展示绘图的功能和配置。

- 添加一个 EMA（指数移动平均线），默认情况下，它会与数据一起绘制。
- 添加一个 WMA（移动平均线加权），配置在子图绘制（即使没有意义）。
- 添加一个 StochasticSlow（慢速随机指标），不更改默认设置。
- 添加一个 MACD，不更改默认设置。
- 添加一个ATR，更改默认设置以避免绘图。
- 添加一个 RSI，不更改默认设置。
- 在 RSI 上添加一个 SMA 指标，不更改默认设置，且与RSI一起绘制。

在策略的 `__init__` 方法中添加的所有内容：

```python
# Indicators for the plotting show
bt.indicators.ExponentialMovingAverage(self.datas[0], period=25)
bt.indicators.WeightedMovingAverage(self.datas[0], period=25).subplot = True
bt.indicators.StochasticSlow(self.datas[0])
bt.indicators.MACDHisto(self.datas[0])
rsi = bt.indicators.RSI(self.datas[0])
bt.indicators.SmoothedMovingAverage(rsi, period=10)
bt.indicators.ATR(self.datas[0]).plot = False
```

即使将指标没有赋值到策略成员变量（如`self.sma = MovingAverageSimple…`），它们也会被注册到策略中，成为图表的一部分。

示例中，只有RSI被添加到临时变量rsi中，其目的是要在其上创建一个 SmoothedMovingAverage。

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import datetime  # For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt

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

        # Indicators for the plotting show
        bt.indicators.ExponentialMovingAverage(self.datas[0], period=25)
        bt.indicators.WeightedMovingAverage(self.datas[0], period=25,
                                            subplot=True)
        bt.indicators.StochasticSlow(self.datas[0])
        bt.indicators.MACDHisto(self.datas[0])
        rsi = bt.indicators.RSI(self.datas[0])
        bt.indicators.SmoothedMovingAverage(rsi, period=10)
        bt.indicators.ATR(self.datas[0], plot=False)

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

    # Plot the result
    cerebro.plot()
```

执行后的输出：

```
Starting Portfolio Value: 1000.00
2000-02-18, Close, 27.61
2000-02-22, Close, 27.97
2000-02-22, BUY CREATE, 27.97
2000-02-23, BUY EXECUTED, Size 10, Price: 28.38, Cost: 283.80, Commission 0.00
2000-02-23, Close, 29.73
...
...
...
2000-12-21, BUY CREATE, 27.82
2000-12-22, BUY EXECUTED, Size 10, Price: 28.65, Cost: 286.50, Commission 0.00
2000-12-22, Close, 30.06
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
2000-12-29, SELL CREATE, 27.41
Final Portfolio Value: 981.00
```

因为交易逻辑没有修改，故而结果和上节一样，图表如下：

![](https://www.backtrader.com/docu/quickstart/quickstart10.png)
