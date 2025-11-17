---
title: "第一个策略"
weight: 4
---

# 第一个策略

本节我们将学习如何开发策略。

第一个策略不涉及交易，我们将通过它打印每一天（bar）的 "收盘价（Close）" 。

策略类（Strategy）继承自 `bt.Strategy`。

```python
class TestStrategy(bt.Strategy):
    def __init__(self):
        pass
    def next(self):
        pass
```

它最重要的两个方法是 `__init__`（策略初始化）和 `next`（每个 OHLC，即bar，执行一次该方法）。

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

如上代码中的 `self.datas[0].close` 访问的就是 `cerebro.broker.adddata` 方法添加的第一个数据源 DataFeed。

数据列表（`self.datas`）是一个标准的Python列表，如添加多个数据源，数据按插入顺序存储。列表中的第一个数据项`self.datas[0]`是默认交易数据，用于同步所有策略元素（它作为系统时钟）。

`self.datas` 的列表元素 data 底层类是 DataSeries，它有别名访问众所周知的 OHLC（开盘、高、低、收盘）每日值。

```python
open = data.open
low = data.low
high = data.high
close = data.close
```

为了便于使用，我们将其赋值到 `self.dataclose`，这将简化打印逻辑的创建。

```python
self.dataclose = self.datas[0].close
```

接着，在 `next` 方法中打印 `self.dataclose[0]` 最新收盘价即可。策略的 `next` 方法会在每个新的bar上调用，使用系统时钟（即`self.datas[0]`）作为参考。

```python
def next(self):
    self.log('Close, %.2f' % self.dataclose[0])
```

我们有了策略类 `TestStrategy`，还要通过 `cerebro.addstrategy` 将其添加交易系统中。

```python
cerebro.addstrategy(TestStrategy)
```

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
...
...
...
2000-12-26, Close, 29.17
2000-12-27, Close, 28.94
2000-12-28, Close, 29.29
2000-12-29, Close, 27.41
Final Portfolio Value: 100000.00
```
