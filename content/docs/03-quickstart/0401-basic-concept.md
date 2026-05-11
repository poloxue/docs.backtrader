---
title: "基础概念"
weight: 5
---

# 基础概念

在动手前，需要先理解两个关键概念：Line（线）和索引 0（Index 0）。

## Line

在 Backtrader 中，几乎一切都由 Line 构成。无论是价格数据、指标还是策略内部变量，都以 Line 的形式存在。

你可以把 Line 理解为一条随时间变化的数据序列，由一系列时间点上的数据构成，例如收盘价随时间变化的轨迹。

一个典型的行情数据源（DataFeed）包含以下关键数据点：

- 开盘价（Open）
- 最高价（High）
- 最低价（Low）
- 收盘价（Close）
- 成交量（Volume）
- 未平仓量（OpenInterest）

沿着时间轴看，每个数据点各自形成一条独立的线：所有收盘价构成 Close Line，所有开盘价构成 Open Line。因此一个数据源包含 6 条线，再加上用于标识时间的 DateTime，一共是 7 条线。

在指标（Indicator）中也是如此。简单移动平均线（SMA）根据收盘价计算平均值序列，这个序列同样是一条 Line。布林带、RSI、MACD 等指标都会在内部生成若干条 Line。

在 Backtrader 中，Line 是一切的基础结构单位：

- 数据 → Line
- 指标 → Line 的运算结果
- 策略 → 对多条 Line 的逻辑组合与判断

## 索引0

理解 Line 之后，另一个核心概念是索引（Index）。在 Python 中，`[0]` 通常表示第一个元素，`[-1]` 表示最后一个元素。但 Backtrader 中有所不同：

- 索引 0（`[0]`）表示当前时刻的值
- 索引 -1（`[-1]`）表示上一个时间点的值
- 索引 -2（`[-2]`）表示再上一个时间点的值，以此类推

例如，假设在策略初始化阶段创建了一个简单移动平均线（SMA）：

```python
self.sma = bt.indicators.SimpleMovingAverage(self.data.close, period=15)
```

当前的 SMA 值是：

```python
current_value = self.sma[0]
```

上一个时间点的 SMA 值：

```python
previous_value = self.sma[-1]
```

继续往前推：

```python
two_bars_ago = self.sma[-2]
```

这样，我们就可以方便地比较过去几个时间点，判断趋势变化、触发信号等。

```python
# 收盘价刚刚从下方突破均线，则买入
if self.data.close[0] > self.sma[0] and self.data.close[-1] <= self.sma[-1]:
    self.buy()
```

这段逻辑就是一个典型的 "均线突破买入信号"。通过索引操作，我们无需关心当前是哪一天或第几根K线，只需直接比较“现在”和“过去”的数值即可。

## Line 和索引的应用

理解了 Line 和索引 0，你就掌握了 Backtrader 的”语言”。编写指标或策略逻辑时，会经常看到类似这样的写法：

```python
self.rsi = bt.indicators.RSI(self.data.close)
```

RSI 指标生成了一条新的 Line。

```python
if self.rsi[0] > 70:
    self.sell()
```

我们通过 rsi[0] 访问当前 RSI 值，当 RSI 超过 70 时，执行卖出操作。

无论是移动平均、布林带、RSI，还是自定义指标，最终都是 `Line + 索引访问` 的形式。
