---
title: "使用指标"
weight: 1
---

# 使用指标

指标可以在平台中的两个地方使用：

- 策略内部
- 其他指标内部

## 指标在操作中的使用

在策略中，指标总是在 `__init__` 中实例化，在 `next` 中使用或检查指标值。一个重要原则是：在 `__init__` 中声明的任何指标（或派生值）都会在调用 `next` 前预先计算。

了解两种模式的差异。

## `__init__` vs `next`

在 `__init__` 中，对线条对象的操作都会生成新的线条对象。在 `next` 中，对线条对象的操作则生成常规的 Python 类型，如浮点数和布尔值。

例如 `__init__` 中的操作：

```python
hilo_diff = self.data.high - self.data.low
```

变量 `hilo_diff` 持有线条对象的引用，该对象在调用 `next` 前预先计算，可通过标准数组索引 `[]` 访问。

它包含数据源中每个条的高低差值。

这在混合简单线条（如 `self.data` 数据源中的线条）和复杂线条（如指标）时也有效：

```python
sma = bt.SimpleMovingAverage(self.data.close)
close_sma_diff = self.data.close - sma
```

现在 `close_sma_diff` 同样包含一个线条对象。

使用逻辑运算符：

```python
close_over_sma = self.data.close > sma
```

现在生成的线条对象将包含一个布尔数组。

在 `next` 中，一个操作（逻辑运算符）：

```python
close_over_sma = self.data.close > self.sma
```

使用等效数组（基于索引 0 的表示法）：

```python
close_over_sma = self.data.close[0] > self.sma[0]
```

这种情况下，`close_over_sma` 生成一个布尔值，即比较 `self.data.close[0]` 和 `self.sma[0]` 两个浮点数的结果。

## 为什么选择 `__init__` vs `next`

逻辑简化和易用性是关键。可以在 `__init__` 中声明计算和大部分相关逻辑，让 `next` 中的实际操作逻辑保持最小。

还有一个附带好处：速度（得益于预先计算）。

一个完整的示例，在 `__init__` 中生成一个买入信号：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):
        sma1 = btind.SimpleMovingAverage(self.data)
        ema1 = btind.ExponentialMovingAverage()

        close_over_sma = self.data.close > sma1
        close_over_ema = self.data.close > ema1
        sma_ema_diff = sma1 - ema1

        buy_sig = bt.And(close_over_sma, close_over_ema, sma_ema_diff > 0)

    def next(self):
        if buy_sig:
            self.buy()
```

Python 的 `and` 运算符无法被重载，因此平台定义了自己的 `And`。其他构造如 `Or` 和 `If` 也是如此。

“声明式”方法在 `__init__` 期间让 `next`（策略实际工作的地方）保持精简。此外，这也加快了执行速度。

当逻辑变得复杂并涉及多个操作时，最好将其封装在指标内。

## 一些注意事项

在上述示例中，与其他平台相比，backtrader 已简化了两件事：

声明的指标既不需要父参数（如创建它们的策略），也无需调用注册方法或函数。但策略仍会自动触发指标及操作生成的线条对象的计算（如 `sma - ema`）。

`ExponentialMovingAverage` 实例化时没有传入 `self.data`，这是有意为之。如果不传数据，父对象（即策略）的第一个数据会自动传入。

## 指标绘图

首先要强调的是：声明的指标会自动绘制（如果调用了 `cerebro.plot`）。操作生成的线条对象不会绘制（如 `close_over_sma = self.data.close > self.sma`）。

可以用辅助的 `LinePlotterIndicator` 来绘制这些操作，用法如下：

```python
close_over_sma = self.data.close > self.sma
LinePlotterIndicator(close_over_sma, name='Close_over_SMA')
```

`name` 参数为指标的单条线命名。

## 控制绘图

在开发指标期间，可以添加一个 `plotinfo` 声明。它可以是一个包含两元素的元组的元组、一个字典或一个有序字典。看起来像这样：

```python
class MyIndicator(bt.Indicator):
    ...
    plotinfo = dict(subplot=False)
    ...
```

稍后可以访问或设置该值（如需）：

```python
myind = MyIndicator(self.data, someparam=value)
myind.plotinfo.subplot = True
```

也可以在实例化时设置该值：

```python
myind = MyIndicator(self.data, someparams=value, subplot=True)
```

`subplot=True` 将传递给指标的（在幕后实例化的）成员变量 `plotinfo`。

`plotinfo` 提供以下参数来控制绘图行为：

参数名            | 默认值       | 描述
----------------- | ------------ | ---------------
`plot`            | True         | 是否绘制指标。
`subplot`         | True         | 是否在不同窗口中绘制指标。对于移动平均线等指标，默认值更改为 False。
`plotname`        | ''           | 设置绘图中显示的名称。空值表示将使用指标的规范名称（class.__name__）。这有一些限制，因为 Python 标识符不能使用算术运算符等。例如，指标 `DI+` 将声明如下： `plotinfo = dict(plotname='DI+')`。
`plotabove`       | False        | 指标通常绘制在其操作的数据下方。将其设置为 True 会使指标绘制在数据上方。
`plotlinelabels`  | False        | 适用于“指标”上的“指标”。如果计算 RSI 的简单移动平均线，绘图通常会显示“SimpleMovingAverage”的名称。这是“指标”的名称，而不是实际绘制的线条的名称。如果将值设置为 True，将使用简单移动平均线内线条的实际名称。
`plotymargin`     | 0.0          | 在指标顶部和底部留出的边距量（0.15 -> 15%）。有时 matplotlib 绘图会超出轴的顶部/底部，可能需要一个边距。
`plotyticks`      | []           | 用于控制绘制的 y 轴刻度。如果传递一个空列表，将自动计算“y 刻度”。对于像随机指标这样的东西，设置为已知的行业标准可能有意义：[20.0, 50.0, 80.0]。某些指标提供 `upperband` 和 `lowerband` 等参数，实际上用于操纵 y 刻度。
`plothlines`      | []           | 用于控制沿指标轴绘制的水平线。如果传递一个空列表，则不会绘制水平线。对于像随机指标这样的东西，绘制已知的行业标准线可能有意义：[20.0, 80.0]。某些指标提供 `upperband` 和 `lowerband` 等参数，实际上用于操纵水平线。
`plotyhlines`     | []           | 用于同时控制 `plotyticks` 和 `plothlines`，使用一个单一参数。
`plotforce`       | False        | 如果由于某种原因你认为某个指标应该绘制但未绘制……请最后设置为 True。
