---
title: "线 Line"
weight: 3
---

# 线 Line

在 `Backtrader` 中，许多对象（如数据源和指标）都会生成 `Line` 对象。每个 `Line` 代表一个时间序列数据，可以是价格、指标或其他数据。我们的策略逻辑基本都离不开操作 `Line` 对象。

## **线的访问**

- **访问数据源中的线 `Line`**

数据源中包含了多个 `Line`，如 `close`、`open`、`high`、`low` ，通过 `self.data` 来访问这些线：

```python
class MyStrategy(bt.Strategy):
   def __init__(self):
       self.close_line = self.data.close  # 访问收盘价线
```

- **访问指标的线**：

指标对象也会生成 `Line`，如 `SimpleMovingAverage` 的 `sma`，通过 `self.movav.lines.sma` 来访问它的 `Line`。

```python
class MyStrategy(bt.Strategy):
   def __init__(self):
       self.movav = btind.SimpleMovingAverage(self.data, period=20)

   def next(self):
       if self.movav.lines.sma[0] > self.data.lines.close[0]:
           print('移动平均大于收盘价')
```

3. **访问线的快捷方式**：
   - 你可以使用简短的语法来访问线。例如，`self.data.close` 实际上是 `self.data.lines.close` 的快捷方式。

通过这些方式，你可以在策略中访问和操作不同的数据线，执行各种分析和交易决策。

## **线的声明**

在 `Backtrader` 中，**线** 是指标和策略中非常重要的部分。每个自定义指标都需要声明自己的线，以便在策略中使用。

### **如何声明线**

1. **在自定义指标中声明线**：
   - 当你创建一个自定义指标时，需要通过 `lines` 属性声明该指标会输出的线。通常使用元组来声明，元组中的每个元素表示一个线的名称。
   - 例如，在 `SimpleMovingAverage` 指标中，我们声明了一个名为 `sma` 的线：
     ```python
     class SimpleMovingAverage(bt.Indicator):
         lines = ('sma',)  # 声明一个名为 sma 的线
         
         def __init__(self):
             self.lines.sma = self.data.close(-1)  # 计算并赋值给 sma 线
     ```

2. **声明多条线**：
   - 如果你希望一个指标输出多条线，只需在 `lines` 中列出多个元素：
     ```python
     class MyIndicator(bt.Indicator):
         lines = ('sma1', 'sma2')  # 声明两个线：sma1 和 sma2
         
         def __init__(self):
             self.lines.sma1 = btind.SimpleMovingAverage(self.data, period=20)
             self.lines.sma2 = btind.SimpleMovingAverage(self.data, period=50)
     ```

3. **注意事项**：
   - 当声明线时，要确保使用元组的形式，即使只有一条线也需要加上逗号：`('sma',)`。
   - 如果你没有为指标声明线，`Backtrader` 会默认处理，但明确声明会让代码更加清晰。

通过这种方式，指标的计算结果会被保存在线中，策略可以直接访问这些线进行操作。


## **访问数据源中的线**

在 `Backtrader` 中，数据源的线可以直接访问，避免了多次重复写 `lines`，让代码更简洁自然。

### **如何访问数据源中的线**

1. **直接访问数据源中的线**：
   - 如果你的数据源包含例如收盘价（`close`）、开盘价（`open`）等线，可以直接通过 `self.data` 访问：
     ```python
     class MyStrategy(bt.Strategy):
         def next(self):
             if self.data.close[0] > 30.0:  # 访问收盘价
                 print("收盘价大于 30")
     ```
   - 在上面的例子中，`self.data.close` 直接引用了数据源的 `close` 线。

2. **避免使用 `lines` 显式声明**：
   - 当访问数据源的线时，通常无需显式使用 `lines`，例如：
     ```python
     if self.data.close[0] > 30.0:  # 直接访问收盘价线
     ```
   - 相比于 `self.data.lines.close[0]`，这种写法更加简洁和自然。

3. **使用自定义数据源时**：
   - 在自定义数据源（例如从 CSV 文件加载数据）时，你也可以像普通数据源一样，直接通过 `self.data` 访问其线：
     ```python
     data = btfeeds.BacktraderCSVData(dataname='mydata.csv')
     ...
     class MyStrategy(bt.Strategy):
         def next(self):
             if self.data.close[0] > 30.0:
                 print("收盘价大于 30")
     ```

通过这种方式，访问数据源的线变得非常直观和简洁，减少了代码的复杂性。

继续输出下一段：

---

## **线的长度**

在 `Backtrader` 中，每条线都包含一个动态增长的点集合。你可以随时获取线的长度，以了解数据的处理情况。

### **如何获取线的长度：**

1. **使用 `len()` 函数**：
   - 你可以使用标准的 Python `len()` 函数来获取线的长度，这将返回已经处理的数据点数。
     ```python
     class MyStrategy(bt.Strategy):
         def next(self):
             length = len(self.data)
             print(f"数据源的线长: {length}")
     ```

2. **使用 `buflen` 属性**：
   - `buflen` 返回数据源可用的总数据条数，也就是数据源在加载时的总长度。
     ```python
     class MyStrategy(bt.Strategy):
         def next(self):
             buflen = self.data.buflen
             print(f"数据源的总长度: {buflen}")
     ```
   - 这对于实时数据源很有用，能够显示数据源的实际可用条数。

3. **`len()` 和 `buflen` 的区别**：
   - `len()` 返回已处理的数据条数，即实际运行到当前时间点的数据长度。
   - `buflen` 返回的是数据源加载的总条数，即所有加载的数据条数。

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

子类不仅会继承父类的参数，还会继承父类定义的线。这样可以使得你在多个策略类中复用线的计算结果。

如果多个父类定义了相同名称的线，子类只会继承一个版本的线，因此要避免同名的线定义冲突。


