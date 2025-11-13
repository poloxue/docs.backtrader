---
title: "开始交易"
weight: 6
---

# 开始交易

本节，我们将演示一个简单的策略，让策略执行交易操作，如果出现连续两个交易日下跌即 - 买入！买入！买入！

我们将基于上节的策略类 TestStrategy 继续开发，策略逻辑部分要在 `next` 方法中实现。

```python
class TestStrategy(bt.Strategy):
    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close

    def next(self):
        self.log('Close, %.2f' % self.dataclose[0])
```

首先是买入条件，如何判断两日连续下跌呢？

简单而言，就是 `close[0] < close[-1]` 和 `close[-1] < close[-2]`，即当前收盘价小于昨日收盘价，昨日收盘价小于大前日收盘价。

```python
self.dataclose[0] < self.dataclose[-1] and self.dataclose[-1] < self.dataclose[-2]
```

而买入操作使用 `self.buy` 皆可。

```python
self.buy()
```

默认情况下，如果存在多个数据源，`self.buy` 买入的是第一个数据资产。

## 完整示例

```python
import datetime  # For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt

class TestStrategy(bt.Strategy):
    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close

    def next(self):
        self.log('Close, %.2f' % self.dataclose[0])

        if self.dataclose[0] < self.dataclose[-1]:
            if self.dataclose[-1] < self.dataclose[-2]:
                self.log('BUY CREATE, %.2f' % self.dataclose[0])
                self.buy()

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.addstrategy(TestStrategy)

    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    datapath = os.path.join(modpath, './orcl-1995-2014.txt')

    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        fromdate=datetime.datetime(2000, 1, 1),
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    cerebro.adddata(data)
    cerebro.broker.setcash(100000.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

输出：

```
Starting Portfolio Value: 100000.00
2000-01-03, Close, 27.85
2000-01-04, Close, 25.39
2000-01-05, Close, 24.05
2000-01-05, BUY CREATE, 24.05
2000-01-06, Close, 22.63
2000-01-06, BUY CREATE, 22.63
2000-01-07, Close, 24.37
...
...
...
2000-12-20, BUY CREATE, 26.88
2000-12-21, Close, 27.82
2000-12-22, Close, 30.06
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-27, BUY CREATE, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
Final Portfolio Value: 99725.08
```

多个“买入”创建订单被发出，我们的组合价值减少了。显然缺少了一些重要的内容，订单已创建，但尚未确定其执行时间和价格。

下个示例将监听订单状态通知来完善这个过程。
