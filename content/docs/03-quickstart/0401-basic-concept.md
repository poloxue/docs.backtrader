---
title: "基础概念"
weight: 5
---

# 基础概念

在正式动手前，我们需要先理解两个极其关键的概念：Line（线） 和 索引 0（Index 0）。

## Line

在 Backtrader 的世界中，几乎一切都由 线（Line） 构成。无论是价格数据、指标还是策略内部变量，它们都以 Line 的形式存在。

你可以把 Line 理解为一条随时间变化的数据序列，就像价格走势图上的那条曲线。一条 Line 是由一系列点（即时间序列数据）构成的，例如收盘价随时间变化的轨迹。

对于一个典型的行情数据源（DataFeed），每天包含以下几个关键数据点：

- 开盘价（Open）
- 最高价（High）
- 最低价（Low）
- 收盘价（Close）
- 成交量（Volume）
- 未平仓量（OpenInterest）

沿着时间轴看，这些数据点各自形成一条独立的“线”：如所有的收盘价构成了 Close Line，所有的开盘价构成了 Open Line。

例如，沿时间轴上的“开盘价”形成了一条线（Line）。因此，一个数据源通常包含6条线。如果再考虑“日期时间” (DateTime)（作为单个点的实际参考），就可以得到7条线（Line）。

因此，一个完整的数据源通常包含 6 条线。如果再加上用于标识时间的 DateTime，那么一共就是 7 条线。

在指标（Indicator）中，这一概念同样适用。如简单移动平均线（SMA） 会根据收盘价计算一个平均值序列，这个序列同样是一条 Line。再如布林带、RSI、MACD 等指标，都会在内部生成若干条 Line，分别表示不同的计算结果。

在 Backtrader 中，Line 是一切的基础结构单位。

```bash
数据 → Line<br/>
指标 → Line 的运算结果<br/>
策略 → 对多条 Line 的逻辑组合与判断<br/>
```

## 索引0

理解 Line 之后，另一个必须掌握的核心概念是 索引（Index）。在 Python 中，索引 [0] 通常表示第一个元素，而 [-1] 表示最后一个元素。但 Backtrader 中，索引的语义稍有不同：

- 索引 0（[0]）表示当前时刻的值
- 索引 -1（[-1]）表示上一个时间点的值
- 索引 -2（[-2]）表示再上一个时间点的值

以此类推。

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

这样，我们就可方便地比较过去几个时间点，判断趋势变化、触发信号等。

```python
# 收盘价刚刚从下方突破均线，则买入
if self.data.close[0] > self.sma[0] and self.data.close[-1] <= self.sma[-1]:
    self.buy()
```

这段逻辑就是一个典型的 "均线突破买入信号"。通过索引操作，我们无需关心当前是哪一天或第几根K线，只需直接比较“现在”和“过去”的数值即可。

## Line 和索引的应用

理解了 Line 和 索引 0，你就掌握了 Backtrader 的“语言”。当编写指标或策略逻辑时，就会经常看到类似这样的写法：

```python
self.rsi = bt.indicators.RSI(self.data.close)
```

RSI 指标生成了一条新的 Line。

```python
if self.rsi[0] > 70:
    self.sell()
```

我们通过 rsi[0] 访问当前 RSI 值，当 RSI 超过 70 时，执行卖出操作。

无论是移动平均、布林带、RSI，还是你的自定义指标，最终都是 `Line + 索引访问` 的形式。
