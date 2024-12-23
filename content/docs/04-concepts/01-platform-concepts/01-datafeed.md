---
title: "数据源 DataFeed"
weight: 1
---

# **数据源 - 配置与使用**

本节介绍 `backtrader` 中数据源的配置与使用，同时也包含了一些数据访问的技巧。

## 数据配置

在 `Backtrader` 中，数据源 `DataFeed` 通过 `Celebro` 配置。

配置代码：

```python
cerebro = bt.Cerebro()

data = btfeeds.MyFeed(...)
cerebro.adddata(data)
cerebro.addstrategy(MyStrategy, period=30)
```

通过 `cerebro.adddata` 将 `DataFeed` 添加到系统中。我们无需关心系统是如何接收 `DataFeed` 的。

## 使用方法

策略中，我们通过 `self.datas` 数组即可访问数据。我们看一个简单示例，如我们实现策略。

示例如下：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.datas[0], period=self.params.period)
```

通过 `self.datas[0]` 即可访问数据。

示例中有两个注意点：

- 策略的 `__init__` 方法无需接收 `*args` 或 `**kwargs`；
- `self.datas` 是一个包含 `DataFeed` 的数组，至少包含一个数据源，否则会出现异常；

一旦数据源被添加到系统中，我们在策略实现就可以按它们添加的顺序访问到每个数据源。

```python
cerebro.adddata(data0)
cerebro.adddata(data1)
```

在策略类访问：

```python
self.datas[0] # data0
self.datas[1] # data1
```

## **快捷访问**

数据源也可通过快捷方式轻松访问，`self.datas` 数组中的每个元素都可以通过自动生成的成员变量进行访问：

对应规则：

- `self.data` 对应的是 `self.datas[0]`
- `self.dataX` 对应的是 `self.datas[X]`

示例：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.params.period)
```

示例中的 `self.data` 是 `self.datas[0]` 的快捷方式，即访问的就是第一个数据源。如果你添加了多个数据源，就可以通过 `self.data1` 访问 `self.datas[1]`，`self.data2` 访问 `self.datas[2]`，依次类推。

## **省略数据源**

上面的示例还可以进一步简化。

在调用 `SimpleMovingAverage` 时，我们可以完全省略 `self.data`，`backtrader` 会自动选择数据源：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(period=self.params.period)
```

在这个简化版本中，`SimpleMovingAverage` 没有显式传递 `self.data`。

如果省略了 `self.data`，`SimpleMovingAverage` 会默认选择第一个数据源（即 `self.data` 或 `self.datas[0]`）作为输入。

好的，我将按照步骤来简化表达，这样更清晰易懂：

---

## **数据延伸**

在 `Backtrader` 中，**不仅数据源**可以作为输入，**指标和计算结果**也能作为数据提供给策略。

通过一个示例说明吧。

将数据源作为输入计算指标，如：

```python
sma1 = btind.SimpleMovingAverage(self.datas[0], period=self.p.period1)
```

`self.datas[0]` 是第一个数据源，它传递给 `SimpleMovingAverage` 进行计算。

基于计算出的指标生成新的变量，如：

```python
sma2 = btind.SimpleMovingAverage(sma1, period=self.p.period2)
```

还可以在指标之间进行运算，将操作结果作为新的变量：

```python
 diff = sma2 - sma1 + self.data.close
 ```

操作结果可以像数据源一样传递给下一个指标计算函数：

 ```python
 sma3 = btind.SimpleMovingAverage(diff, period=self.p.period3)
 greater = sma3 > sma1
 sma4 = btind.SimpleMovingAverage(greater, period=self.p.period4)
 ```

如上的步骤中，我们不断将计算的中间结果作为新的数据源，传递给后续指标进一步计算。

