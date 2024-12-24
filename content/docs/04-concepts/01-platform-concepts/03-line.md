---
title: "线 Line 类"
weight: 3
---

# 线 Line 类

在 `Backtrader` 中，许多对象都会生成 `Line` 对象，而每个 `Line` 代表的是一个时间序列数据，可以是价格、指标或其他数据。策略逻辑基本都离不开操作 `Line` 对象。

## **`Line` 的访问**

### **数据源中的 `Line`**

数据源中包含了多个 `Line`，如 `close`、`open`、`high`、`low` ，通过 `self.data.lines` 访问它们。

```python
class MyStrategy(bt.Strategy):
   def __init__(self):
       self.close_line = self.data.lines.close  # 访问收盘价线
```

### **指标中的 `Line`**

指标同样会生成 `Line`，如 `SimpleMovingAverage` 的 `sma`，通过 `self.movav.lines.sma` 访问。

```python
class MyStrategy(bt.Strategy):
   def __init__(self):
       self.movav = btind.SimpleMovingAverage(self.data, period=20)

   def next(self):
       if self.movav.lines.sma[0] > self.data.lines.close[0]:
           print('移动平均大于收盘价')
```

### **访问线的快捷方式**

前面的语法和我们平时使用的不一样，因为我们平时都是通过简写访问，如 `self.data.close` 实际上是 `self.data.lines.close` 的快捷方式。

`Backtrader` 提供了多种简化访问 `Line` 的方式：

- `xxx.lines` 可简写 `xxx.l`；
- `xxx.lines.name` 可简写 `xxx.lines_name`；
- `xxx.lines[0]` 可简写为 `xxx`
- `xxx.lines[X]` 可简写为 `xxx.lineX`；

还有如：

- `self.data_name` 等同于 `self.data.lines.name`；
- 对于有编号的 `data`，如 `self.data1_name`，等同于 `self.data1.lines.name`。
- 还有 `self.dataX_Y` 等同于 `self.data[X].lines[Y]`；

一些常用的简化写法，通过属性访问这些 `Line`：

- `self.data.close` 或 `self.data_close` 直接访问 `close` 数据行。
- `self.movav.sma` 或 `self.movav_sma` 直接访问 `sma` 数据行。
- `self.movav` 也可直接访问 `sma` 数据行；

这种访问方式很简洁，但不如原始方法清晰，特别是在区分访问的是否 `Line` 时。

## **线的声明**

在 `Backtrader` 中，`Line` 是指标和策略中非常重要的部分。每个自定义指标都要声明自己的线，以便在策略中使用。

### **如何声明线**

如何在自定义指标中声明线？

当你创建一个自定义指标时，需要通过 `lines` 属性声明该指标会输出的线。通常使用元组声明，元组中的每个元素表示一个线的名称。

如在 `SimpleMovingAverage` 指标中，我们声明了一个名为 `sma` 的线：

```python
class SimpleMovingAverage(bt.Indicator):
   lines = ('sma',)  # 声明一个名为 sma 的线
   
   def __init__(self):
       self.lines.sma = self.data.close(-1)  # 计算并赋值给 sma 线
```

如你想声明多条线，如一个包含两条均线的指标，只需在 `lines` 中列出多个元素：

```python
class MyIndicator(bt.Indicator):
   lines = ('sma1', 'sma2')  # 声明两个线：sma1 和 sma2
   
   def __init__(self):
       self.lines.sma1 = btind.SimpleMovingAverage(self.data, period=20)
       self.lines.sma2 = btind.SimpleMovingAverage(self.data, period=50)
```

**注意事项**：当声明 `Line` 时，确保使用元组声明，即使只有一条线也要加上逗号：`('sma',)`。

现在，指标的计算结果会被保存在 `Line` 中，策略可以直接访问这些 `Line` 计算交易逻辑。

---

## **`Line` 的长度**

在 `Backtrader` 中，每条线都包含一个动态增长的点集合。你可以随时获取线的长度，以了解当前的数据处理情况。

### **如何获取线的长度：**

**使用 `len()` 函数**：使用标准 Python `len()` 函数即可获取 `Line` 的长度，这会返回已处理的数据点数。

```python
class MyStrategy(bt.Strategy):
   def next(self):
       length = len(self.data)
       print(f"数据源的线长: {length}")
```

**使用 `buflen` 属性**：`buflen` 返回数据源可用的总数据条数，也就是数据源在加载时的总长度。

```python
class MyStrategy(bt.Strategy):
   def next(self):
       buflen = self.data.buflen
       print(f"数据源的总长度: {buflen}")
```

实盘交易时，对于实时数据源很有用，因为它能让我们显示数据源的实际可用数量。

**`len()` 和 `buflen` 的区别**：

- `len()` 返回已处理的数据条数，即实际运行到当前时间点的数据长度。
- `buflen` 返回的是数据源加载的总条数，即加载的所有数据的长度。

如果两者的值相同，说明所有数据都已经处理完毕。

---

## **线的继承**

`Backtrader` 支持线的继承，我们可以在子类中继承父类的线，且可以在子类中修改。

示例代码：

```python
class BaseIndicator(bt.Indicator):
    lines = ('sma',)

class MyIndicator(BaseIndicator):
    lines = ('sma', 'ema')  # 在子类中继承并扩展线
```

如果多个父类定义了相同名称的线，子类只会继承一个版本的线，因此要避免同名的线定义冲突。


