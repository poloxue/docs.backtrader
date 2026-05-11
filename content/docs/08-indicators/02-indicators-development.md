---
title: "指标开发"
weight: 2
---

# 指标开发

如果要开发任何东西（除了交易策略），那就是自定义指标。在平台中开发它很容易。

开发要点：

- 从 `Indicator` 类（直接或从现有子类）派生一个类；
- 定义它将包含的 `Line`；
- 一个指标至少要有一条线。如果从现有的类派生，线条可能已经定义好了
- 可选地定义可以改变行为的参数
- 可选地提供/自定义一些用于合理绘制指标的元素
- 在 `__init__` 中提供一个完全定义的操作，并绑定（分配）到指标的线条，或者提供 `next` 方法和（可选的）`once` 方法

如果一个指标可以在初始化时通过逻辑或算术操作完全定义，并将结果分配给线条，那就完成了。如果做不到，至少要提供 `next` 方法，在索引 0 处给线条赋值。还可以提供 `once` 方法来优化批处理模式的计算。

## 重要说明：幂等性

指标为每个收到的条生成一个输出。同一个条可能被多次发送，因此操作必须是幂等的。

其背后的理由：

- 同一个条（同一索引）可以多次发送，且值会变化（如收盘价更新）
- 这使得可以“重放”一个日内会话，但使用由 5 分钟条组成的日内数据。

这也使平台能从实时数据源获取值。

## 一个简单（但功能齐全）的指标

可以这样：

```python
class DummyInd(bt.Indicator):
    lines = ('dummyline',)

    params = (('value', 5),)

    def __init__(self):
        self.lines.dummyline = bt.Max(0.0, self.params.value)
```

完成！

该指标始终输出相同的值：0.0 或 `self.params.value`（如果大于 0.0）。

使用 `next` 方法的相同指标：

```python
class DummyInd(bt.Indicator):
    lines = ('dummyline',)

    params = (('value', 5),)

    def next(self):
        self.lines.dummyline[0] = max(0.0, self.params.value)
```

完成！相同行为。

在 `__init__` 版本中，使用 `bt.Max` 将值分配给线条对象 `self.lines.dummyline`。`bt.Max` 返回一个线条对象，它会为每个传入的条自动迭代。如果使用 `max`，赋值将无意义，因为指标得到的是固定值的成员变量，而非线条。在 `next` 中处理浮点值时，可使用标准的 `max` 函数。

回顾一下，`self.lines.dummyline` 是长表示法，可以缩短为：

```python
self.l.dummyline
```

甚至可以缩短为：

```python
self.dummyline
```

只有在代码没有用成员属性遮蔽这个变量时，后者才有可能。

第三个也是最后一个版本提供了一个额外的 `once` 方法来优化计算：

```python
class DummyInd(bt.Indicator):
    lines = ('dummyline',)

    params = (('value', 5),)

    def next(self):
        self.lines.dummyline[0] = max(0.0, self.params.value)

    def once(self, start, end):
        dummy_array = self.lines.dummyline.array

        for i in range(start, end):
            dummy_array[i] = max(0.0, self.params.value)
```

这样更高效，但开发 `once` 方法需要深入研究内部机制。

`__init__` 版本仍然是最好的选择：

- 所有工作都在初始化时完成
- 自动提供 `next` 和 `once`（都已优化，因为 `bt.Max` 已包含它们），无需处理索引和公式

如果需要开发，指标也可以覆盖与 `next` 和 `once` 相关的方法：

- `prenext` 和 `nextstart`
- `preonce` 和 `oncestart`

## 手动/自动最小周期

平台会尽可能自动计算，但有时需要手动指定。

以下是一个简单移动平均线的实现：

```python
class SimpleMovingAverage1(Indicator):
    lines = ('sma',)
    params = (('period', 20),)

    def next(self):
        datasum = math.fsum(self.data.get(size=self.p.period))
        self.lines.sma[0] = datasum / self.p.period
```

虽然看似合理，但平台并不知道最小周期，即使参数名为”period”（名称可能有误导性，有些指标有多个”period”参数，用途各不相同）。

这种情况下，`next` 在第一个条进入时就会被调用，导致崩溃，因为 `get` 无法返回所需的 `self.p.period` 个数据。

解决此问题前，需要考虑：

传递给指标的数据源可能已经携带最小周期。示例 `SimpleMovingAverage` 可以在以下情况下进行：
- 常规数据源：默认最小周期为 1（只等待进入系统的第一个条）。
- 另一个移动平均线：它已有周期。如果是 20，示例移动平均线也是 20，则最终最小周期为 40 条。

实际上内部计算是 39，因为第一个移动平均线生成的条会计入下一个，产生一个重叠条，因此只需 39。其他指标/对象也带有周期。

解决方法如下：

```python
class SimpleMovingAverage1(Indicator):
    lines = ('sma',)
    params = (('period', 20),)

    def __init__(self):
        self.addminperiod(self.params.period)

    def next(self):
        datasum = math.fsum(self.data.get(size=self.p.period))
        self.lines.sma[0] = datasum / self.p.period
```

`addminperiod` 方法告知系统该指标所需的额外周期条数，无论现有最小周期是多少。如果所有计算都使用已向系统传达周期需求的对象，则无需此操作。

一个快速的 MACD 实现带有直方图：

```python
from backtrader.indicators import EMA

class MACD(Indicator):
    lines = ('macd', 'signal', 'histo',)
    params = (('period_me1', 12), ('period_me2', 26), ('period_signal', 9),)

    def __init__(self):
        me1 = EMA(self.data, period=self.p.period_me1)
        me2 = EMA(self.data, period=self.p.period_me2)
        self.l.macd = me1 - me2
        self.l.signal = EMA(self.l.macd, period=self.p.period_signal)
        self.l.histo = self.l.macd - self.l.signal
```

完成！无需考虑最小周期。

EMA 代表指数移动平均线（平台内置别名），已在平台中声明了它的周期需求。指标的命名线条 “macd” 和 “signal” 被分配了已在幕后声明周期的对象。

macd 从 `me1 - me2` 操作中获取周期，me1 和 me2 都是不同周期的指数移动平均线。signal 直接从 macd 上的指数移动平均线获取周期。这个 EMA 也会考虑现有的 macd 周期和自身所需的样本量（`period_signal`）。histo 取 “signal - macd” 两个操作数的最大值。两者都准备好后，histo 就能生成值。

## 一个完整的自定义指标

让我们开发一个简单的自定义指标，它“指示”一个移动平均线（可以通过参数修改）是否在给定数据之上：

```python
import backtrader as bt
import backtrader.indicators as btind

class OverUnderMovAv(bt.Indicator):
    lines = ('overunder',)
    params = dict(period=20, movav=btind.MovAv.Simple)

    def __init__(self):
        movav = self.p.movav(self.data, period=self.p.period)
        self.l.overunder = bt.Cmp(movav, self.data)
```

完成！

如果平均线在数据之上，指标值为 “1”；在数据之下则为 “-1”。对于常规数据源，收盘价比较会生成 1 或 -1。

虽然绘图部分会自动处理，但为了让指标更友好易用，可以添加以下内容：

```python
import backtrader as bt
import backtrader.indicators as btind

class OverUnderMovAv(bt.Indicator):
    lines = ('overunder',)
    params = dict(period=20, movav=btind.MovAv.Simple)

    plotinfo = dict(


        # 在 1 和 -1 之上和之下添加额外的边距
        plotymargin=0.15,

        # 绘制参考水平线在 1.0 和 -1.0
        plothlines=[1.0, -1.0],

        # 简化 y 轴刻度为 1.0 和 -1.0
        plotyticks=[1.0, -1.0])

    # 使用虚线样式绘制 "overunder" 线（唯一的一个）
    # ls 代表线条样式，并直接传递给 matplotlib
    plotlines = dict(overunder=dict(ls='--'))

    def _plotlabel(self):
        # 此方法返回将在绘图上显示的标签列表
        # 在指标名称之后

        # 周期必须始终存在
        plabels = [self.p.period]

        # 如果不是默认移动平均线，只放移动平均线
        plabels += [self.p.movav] * self.p.notdefault('movav')

        return plabels

    def __init__(self):
        movav = self.p.movav(self.data, period=self.p.period)
        self.l.overunder = bt.Cmp(movav, self.data)
```

完成！

这个指标将具有 1 和 -1 的值，并且在绘图中有良好的表现。
