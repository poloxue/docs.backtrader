---
title: "平台概念"
weight: 1
---

# 平台概念

先介绍一些 `backtrader` 这个工具平台上的基础概念，希望能让你更好地了解和使用平台。

---

## **开始之前**

先导入必要的模块：

```python
import backtrader as bt
import backtrader.indicators as btind
import backtrader.feeds as btfeeds
```

如下方法也可以访问子模块（如指标和数据源）：

```python
import backtrader as bt

thefeed = bt.feeds.OneOfTheFeeds(...)
theind = bt.indicators.SimpleMovingAverage(...)
```

#### 运算符，自然构造

为了实现“易用”目标，平台允许（在 Python 的约束范围内）使用运算符。为了进一步增强此目标，运算符的使用被分为两个阶段。

##### 阶段 1 - 运算符创建对象

即使不是明确地为此目的，之前的一个示例已经看到这一点。在像指标和策略这样的对象的初始化阶段（`__init__` 方法），运算符创建可以被操作、分配或保留以供策略逻辑的评估阶段稍后使用的对象。

再次查看 `SimpleMovingAverage` 的潜在实现，进一步分解为步骤。

在 `SimpleMovingAverage` 指标的 `__init__` 方法内部的代码可能如下：

```python
def __init__(self):
    # 求和 N 期值 - datasum 现在是一个 *Lines* 对象
    # 在使用运算符 [] 和索引 0 查询时
    # 返回当前和

    datasum = btind.SumN(self.data, period=self.params.period)

    # datasum（尽管是单行，但作为 *Lines* 对象）可以
    # 像在这种情况下那样自然地除以 int/float。
    # 实际上它可以被另一个 *Lines* 对象除。
    # 操作返回一个对象，分配给 "av"，该对象再次
    # 在使用 [0] 查询时返回当前的平均值

    av = datasum / self.params.period

    # av *Lines* 对象可以自然地分配给命名的
    # 线该指标交付。使用此
    # 指标的其他对象将可以直接访问计算结果

    self.line.sma = av
```

在策略的初始化中显示一个更完整的用例：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma = btind.SimpleMovingAverage(self.data, period=20)

        close_over_sma = self.data.close > sma
        sma_dist_to_high = self.data.high - sma

        sma_dist_small = sma_dist_to_high < 3.5

        # 不幸的是，"and" 不能在 Python 中被覆盖，
        # 因为它是语言构造而不是运算符，因此平台
        # 提供了一个函数来模拟它

        sell_sig = bt.And(close_over_sma, sma_dist_small)
```

在上述操作完成后，`sell_sig` 是一个线对象，可以在策略的逻辑中稍后使用，以指示条件是否满足。

##### 阶段 2 - 运算符忠于自然

让我们首先记住，一个策略有一个 `next` 方法，系统处理每个条时调用该方法。这是运算符实际上处于阶段 2 模式的地方。基于前面的示例：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        self.sma = sma = btind.SimpleMovingAverage(self.data, period=20)

        close_over_sma = self.data.close > sma
        self.sma_dist_to_high = self.data.high - sma

        sma_dist_small = sma_dist_to_high < 3.5

        # 不幸的是，"and" 不能在 Python 中被覆盖，
        # 因为它是语言构造而不是运算符，因此平台
        # 提供了一个函数来模拟它

        self.sell_sig = bt.And(close_over_sma, sma_dist_small)

    def next(self):

        # 尽管这看起来不像是一个“运算符”，但实际上是
        # 在测试对象是否响应 True/False

        if self.sma > 30.0:
            print('简单移动平均线大于 30.0')

        if self.sma > self.data.close:
            print('简单移动平均线高于收盘价')

        if self.sell_sig:  # if sell_sig == True: 也是有效的
            print('卖出信号为真')
        else:
            print('卖出信号为假')

        if self.sma_dist_to_high > 5.0:
            print('简单移动平均线与高点的距离大于 5.0')
```

不是一个非常有用的策略，只是一个示例。在阶段 2 中，运算符返回预期的值（如果测试为真，则为布尔值，如果与浮点数比较，则为浮点数），算术运算也返回预期值。

**注意**

请注意，比较实际上并没有使用 `[]` 操作符。这是为了进一步简化事情。

- `if self.sma > 30.0:` 比较 `self.sma[0]` 和 30.0（第一行和当前值）
- `if self.sma > self.data.close:` 比较 `self.sma[0]` 和 `self.data.close[0]`

#### 一些未覆盖的运算符/函数

Python 不允许覆盖所有内容，因此提供了一些函数来应对这些情况。

**注意**

仅在阶段 1 中使用，用于创建对象以便稍后提供值。

运算符：

- `and` -> `And`
- `or` -> `Or`

逻辑控制：

- `if` -> `If`

函数：

- `any` -> `Any`
- `all` -> `All`
- `cmp` -> `Cmp`
- `max` -> `Max`
- `min` -> `Min`
- `sum` -> `Sum`

`Sum` 实际上使用 `math.fsum` 作为底层操作，因为平台使用浮点数，应用常规求和可能会影响精度。

- `reduce` -> `Reduce`

这些实用运算符/函数对可迭代对象操作。可迭代对象中的元素可以是常规的 Python 数值类型（整数、浮点数等），也可以是具有线的对象。

生成非常愚蠢的买入信号的示例：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma1 = btind.SMA(self.data.close, period=15)
        self.buysig = bt.And(sma1 > self.data.close, sma1 > self.data.high)

    def next(self):
        if self.buysig[0]:
            pass  # 在这里做一些事情
```

显然，如果 `sma1` 高于高点，它必须高于收盘价。但重点是说明 `bt.And` 的使用。

使用 `bt.If`：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma1 = btind.SMA(self.data.close, period=15)
        high_or_low = bt.If(sma1 > self.data.close, self.data.low, self.data.high)
        sma2 = btind.SMA(high_or_low, period=15)
```

分解：

- 在 `data.close` 上生成一个周期为 15 的 `SMA`
- 然后
- `bt.If` 的值大于 `close`，返回 `low`，否则返回 `high`

请记住，在调用 `bt.If` 时没有实际值返回。它返回一个线对象，就像 `SimpleMovingAverage` 一样。

值将在系统运行时计算。

生成的 `bt.If` 线对象然后被馈送到第二个 `SMA`，该 `SMA` 有时使用低价，有时使用高价进行计算。

这些函数也接受数值。同样的示例有一个修改：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma1 = btind.SMA(self.data.close, period=15)
        high_or_30 = bt.If(sma1 > self.data.close, 30.0, self.data.high)
        sma2 = btind.SMA(high_or_30, period=15)
```

现在，第二个移动平均线使用 30.0 或高价进行计算，具体取决于 `sma` 和 `close` 的逻辑状态。

**注意**

值 30 内部被转换为一个伪可迭代对象，该对象总是返回 30。
