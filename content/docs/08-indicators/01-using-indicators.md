---
title: "使用指标"
weight: 1
---

### 使用指标

指标可以在平台中的两个地方使用：

- 策略内部
- 其他指标内部

#### 指标在操作中的使用

在策略中，指标总是在 `__init__` 中实例化。

在 `next` 中使用/检查指标值（或派生值）。

有一个重要的公理需要考虑：

在 `__init__` 中声明的任何指标（或派生值）将在调用 `next` 之前预先计算。

让我们了解操作模式的差异。

#### `__init__` vs `next`

操作如下：

- 在 `__init__` 中涉及到线条对象的任何操作都会生成另一个线条对象。
- 在 `next` 中涉及到线条对象的任何操作都会生成常规的 Python 类型，如浮点数和布尔值。

**在 `__init__` 中：**

例如，`__init__` 中的一个操作：

```python
hilo_diff = self.data.high - self.data.low
```

变量 `hilo_diff` 持有一个线条对象的引用，该对象在调用 `next` 之前预先计算，可以使用标准数组表示法 `[]` 访问。

它显然包含了数据源中每个条的高低差值。

这在混合简单线条（如 `self.data` 数据源中的线条）和复杂线条（如指标）时也有效：

```python
sma = bt.SimpleMovingAverage(self.data.close)
close_sma_diff = self.data.close - sma
```

现在 `close_sma_diff` 再次包含一个线条对象。

使用逻辑运算符：

```python
close_over_sma = self.data.close > sma
```

现在生成的线条对象将包含一个布尔数组。

**在 `next` 中：**

例如，一个操作（逻辑运算符）：

```python
close_over_sma = self.data.close > self.sma
```

使用等效数组（基于索引 0 的表示法）：

```python
close_over_sma = self.data.close[0] > self.sma[0]
```

在这种情况下，`close_over_sma` 生成一个布尔值，这是比较两个浮点值的结果，这些值由应用于 `self.data.close` 和 `self.sma` 的 `[0]` 运算符返回。

#### 为什么选择 `__init__` vs `next`

逻辑简化（以及易用性）是关键。可以在 `__init__` 中声明计算和大部分相关逻辑，将实际操作逻辑保持在 `next` 中的最小化。

实际上还有一个附带好处：速度（由于开始时的预计算）。

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

**注意：**

Python 的 `and` 运算符无法被重载，迫使平台定义其自己的 `And`。同样适用于其他构造如 `Or` 和 `If`。

显然，“声明式”方法在 `__init__` 期间保持 `next`（实际策略工作发生的地方）的膨胀最小化。

（不要忘记，这也有一个加速因素）

**注意：**

当逻辑变得非常复杂并涉及多个操作时，通常最好将其封装在一个指标内。

### 一些注意事项

在上述示例中，与其他平台相比，backtrader 已简化了两件事：

声明的指标既不获取父参数（如它们被创建的策略），也没有调用任何类型的“注册”方法/函数。

尽管如此，策略会触发指标和任何因操作生成的线条对象的计算（如 `sma - ema`）。

`ExponentialMovingAverage` 实例化时没有传入 `self.data`。

这是故意的。如果没有传入数据，父（在本例中是创建它的策略）的第一个数据将在后台自动传递。

### 指标绘图

首先也是最重要的：

声明的指标会自动绘制（如果调用了 `cerebro.plot`）。

操作生成的线条对象不会绘制（如 `close_over_sma = self.data.close > self.sma`）。

有一个辅助的 `LinePlotterIndicator` 可以绘制这些操作，如果需要可以使用以下方法：

```python
close_over_sma = self.data.close > self.sma
LinePlotterIndicator(close_over_sma, name='Close_over_SMA')
```

`name` 参数为此指标持有的单行命名。

### 控制绘图

在开发指标期间，可以添加一个 `plotinfo` 声明。它可以是一个包含两元素的元组的元组、一个字典或一个有序字典。看起来像这样：

```python
class MyIndicator(bt.Indicator):
    ...
    plotinfo = dict(subplot=False)
    ...
```

稍后可以访问（并设置）该值（如果需要）：

```python
myind = MyIndicator(self.data, someparam=value)
myind.plotinfo.subplot = True
```

甚至可以在实例化期间设置该值：

```python
myind = MyIndicator(self.data, someparams=value, subplot=True)
```

`subplot=True` 将传递给指标的（在幕后实例化的）成员变量 `plotinfo`。

`plotinfo` 提供以下参数来控制绘图行为：

- `plot`（默认值：True）：是否绘制指标。
- `subplot`（默认值：True）：是否在不同窗口中绘制指标。对于移动平均线等指标，默认值更改为 False。
- `plotname`（默认值：''）：设置绘图中显示的名称。空值表示将使用指标的规范名称（class.__name__）。这有一些限制，因为 Python 标识符不能使用算术运算符等。例如，指标 `DI+` 将声明如下：

```python
class DIPlus(bt.Indicator):
    plotinfo = dict(plotname='DI+')
```

- `plotabove`（默认值：False）：指标通常绘制在其操作的数据下方。将其设置为 True 会使指标绘制在数据上方。
- `plotlinelabels`（默认值：False）：适用于“指标”上的“指标”。如果计算 RSI 的简单移动平均线，绘图通常会显示“SimpleMovingAverage”的名称。这是“指标”的名称，而不是实际绘制的线条的名称。如果将值设置为 True，将使用简单移动平均线内线条的实际名称。
- `plotymargin`（默认值：0.0）：在指标顶部和底部留出的边距量（0.15 -> 15%）。有时 matplotlib 绘图会超出轴的顶部/底部，可能需要一个边距。
- `plotyticks`（默认值：[]）：用于控制绘制的 y 轴刻度。如果传递一个空列表，将自动计算“y 刻度”。对于像随机指标这样的东西，设置为已知的行业标准可能有意义：[20.0, 50.0, 80.0]。某些指标提供 `upperband` 和 `lowerband` 等参数，实际上用于操纵 y 刻度。
- `plothlines`（默认值：[]）：用于控制沿指标轴绘制的水平线。如果传递一个空列表，则不会绘制水平线。对于像随机指标这样的东西，绘制已知的行业标准线可能有意义：[20.0, 80.0]。某些指标提供 `upperband` 和 `lowerband` 等参数，实际上用于操纵水平线。
- `plotyhlines`（默认值：[]）：用于同时控制 `plotyticks` 和 `plothlines`，使用一个单一参数。
- `plotforce`（默认值：False）：如果由于某种原因您认为某个指标应该绘制但未绘制……请最后设置为 True。
