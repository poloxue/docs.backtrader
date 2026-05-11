---
title: "运算符"
weight: 5
---

# 运算符

在 `Backtrader` 中，运算符不仅用于常规数学运算，还能构建复杂的策略逻辑。自定义运算符让策略的数学和逻辑运算更自然简洁。

## 如何使用运算符

`backtrader` 支持用运算符创建新对象，如在 `__init__` 中通过运算符计算多个指标，得到一个新的操作对象。

```python
class MyStrategy(bt.Strategy):
   def __init__(self):
       sma = btind.SimpleMovingAverage(self.data, period=20)

       # 使用运算符创建新的逻辑对象
       close_over_sma = self.data.close > sma
       sma_dist_to_high = self.data.high - sma
       sma_dist_small = sma_dist_to_high < 3.5

       # 创建卖出信号
       self.sell_sig = bt.And(close_over_sma, sma_dist_small)
```

在 `Line` 对象上使用常规运算符，如加减乘除、大小比较等，简化了策略代码，增强了可读性和可维护性。

注：`backtrader` 的指标计算是自有体系，不是基于 `numpy` 和 `pandas`，因此要单独实现这些运算符。

---

## 一些未覆盖的运算符/函数

Python 中的某些运算符未被覆盖，`backtrader` 提供了专门的函数来模拟逻辑运算，如 `bt.And` 和 `bt.Or` 实现逻辑"与"和"或"。

下面列出这些单独实现的运算符。

### 逻辑运算符

Python 中的 `and` 和 `or` 运算符无法在 `Backtrader` 中覆盖，`backtrader` 提供了 `bt.And` 和 `bt.Or` 来模拟这两个逻辑操作。

```python
self.buy_sig = bt.And(self.data.close > self.sma, self.data.high < 50.0)
self.sell_sig = bt.Or(self.data.close < self.sma, self.data.low > 30.0)
```

### 数学函数

`Backtrader` 也提供了替代 Python 标准库函数的方式，如 `max` 和 `min` 对应 `bt.Max` 和 `bt.Min`。这些函数可用于处理 `Line` 对象：

```python
highest = bt.Max(self.data.high, period=20)  # 获取过去20个周期的最高价
lowest = bt.Min(self.data.low, period=20)    # 获取过去20个周期的最低价
```

### 使用 `bt.If` 模拟条件分支

按条件选择值时，`Backtrader` 提供了 `bt.If` 来模拟条件分支。`bt.If` 类似于 Python 的三元运算符 `x if condition else y`，或者 `numpy` 中的 `where` 函数。

```python
class MyStrategy(bt.Strategy):
   def __init__(self):
       sma1 = btind.SMA(self.data.close, period=15)
       
       # 使用 bt.If 来根据条件选择价格
       high_or_low = bt.If(sma1 > self.data.close, self.data.low, self.data.high)
       sma2 = btind.SMA(high_or_low, period=15)  # 使用选中的值计算新的简单移动平均线
```

### 其他函数

类似地，`any` 对应 `bt.Any`，`all` 对应 `bt.All`，`sum` 对应 `bt.Sum`，`cmp` 对应 `bt.Cmp`，`reduce` 对应 `bt.Reduce`。

这些函数都可用于处理可迭代对象，与 `Line` 对象兼容。

```python
sum_values = bt.Sum(self.data.close, period=10)  # 计算过去10个周期的收盘价总和
```

