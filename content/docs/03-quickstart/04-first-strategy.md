---
title: "第一个策略"
weight: 4
---

# 第一个策略

{{< youtube DfqY81TQdTg >}}

本节我们将学习如何开发策略。

第一个策略不涉及交易，只用来打印每一天（bar）的收盘价。

策略类（Strategy）继承自 `bt.Strategy`。

```python
class TestStrategy(bt.Strategy):
    def __init__(self):
        pass
    def next(self):
        pass
```

它最重要的两个方法是 `__init__`（策略初始化）和 `next`（每个 bar 执行一次）。

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

`self.datas[0].close` 访问的是通过 `cerebro.adddata` 添加的第一个数据源（DataFeed）。

数据列表 `self.datas` 是一个标准 Python 列表，按插入顺序存储。第一个数据 `self.datas[0]` 是默认交易数据，作为系统时钟同步所有策略元素。

`self.datas` 中的元素是 DataSeries 类型，可通过别名访问 OHLC 数据。

```python
open = data.open
low = data.low
high = data.high
close = data.close
```

为了便于使用，我们将其赋值到 `self.dataclose`，简化打印逻辑。

```python
self.dataclose = self.datas[0].close
```

接着在 `next` 方法中打印 `self.dataclose[0]`，即最新收盘价。策略的 `next` 方法会在每个新 bar 上调用，使用系统时钟（`self.datas[0]`）作为参考。

```python
def next(self):
    self.log('Close, %.2f' % self.dataclose[0])
```

有了策略类 `TestStrategy`，还要通过 `cerebro.addstrategy` 将其添加到交易系统中。

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
