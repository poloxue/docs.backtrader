---
title: "参考"
weight: 3
---

# 参考

## Benchmark

```python
class backtrader.observers.Benchmark()
```

该观察器存储策略的回报和作为参考资产的回报，这个参考资产是传递给系统的一个数据。

**参数：**

- `timeframe`（默认：无）：如果为 None，则报告整个回测期间的总回报。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 “TimeFrame.Minutes” 和 60 作为压缩在每小时时间框架上工作。
- `data`（默认：无）：跟踪的参考资产以便进行比较。

**注意**：此数据必须已通过 `adddata`、`resampledata` 或 `replaydata` 添加到 cerebro 实例中。

- `_doprenext`（默认：False）：基准测试将在策略开始运行时进行（即策略的最小周期已达到时）。将其设置为 True 将从数据源的起点记录基准值。
- `firstopen`（默认：False）：保持为 False 确保价值和基准之间的首次比较点从 0% 开始，因为基准不会使用其开盘价。参见 `TimeReturn` 分析器参考以获得参数的完整解释。
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。设置为 True 或 False 以获得特定行为。

记住，在运行的任何时刻都可以通过查看索引 0 处的线条名称来检查当前值。

## Broker

```python
class backtrader.observers.Broker(*args, **kwargs)
```

该观察器跟踪经纪人中的当前现金金额和投资组合价值（包括现金）。

**参数**：无

## Broker - Cash

```python
class backtrader.observers.Cash(*args, **kwargs)
```

该观察器跟踪经纪人中的当前现金金额。

**参数**：无

#### Broker - Value

```python
class backtrader.observers.Value(*args, **kwargs)
```

该观察器跟踪经纪人中的当前投资组合价值，包括现金。

**参数**：

- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。设置为 True 或 False 以获得特定行为。

## BuySell

```python
class backtrader.observers.BuySell(*args, **kwargs)
```

该观察器跟踪单个买入/卖出订单（单个执行）并将在图表上绘制它们，围绕执行价格水平绘制。

**参数**：

- `barplot`（默认：`False`）：在最低点下方绘制买入信号，在最高点上方绘制卖出信号。如果为 `False`，则将在条形的平均执行价格上绘制。
- `bardist`（默认：`0.015` 1.5%）：当 `barplot` 为 `True` 时，与最大值/最小值的距离。

## DrawDown

```python
class backtrader.observers.DrawDown()
```

该观察器跟踪当前的回撤水平（绘图）和最大回撤（不绘图）水平。

**参数**：

- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。设置为 True 或 False 以获得特定行为。

## TimeReturn

```python
class backtrader.observers.TimeReturn()
```

该观察器存储策略的回报。

**参数**：

- `timeframe`（默认：无）：如果为 None，则报告整个回测期间的总回报。传递 `TimeFrame.NoTimeFrame` 以考虑整个数据集而不受时间限制。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 “TimeFrame.Minutes” 和 60 作为压缩在每小时时间框架上工作。
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。设置为 True 或 False 以获得特定行为。

记住，在运行的任何时刻都可以通过查看索引 0 处的线条名称来检查当前值。

## Trades

```python
class backtrader.observers.Trades()
```

该观察器跟踪完整的交易，并在交易关闭时绘制实现的损益水平。

交易在仓位从 0（或跨越 0）变为 X 时开立，然后在回到 0（或反方向跨越 0）时关闭。

**参数**：

- `pnlcomm`（默认：`True`）：显示净利润和亏损，即扣除佣金后的结果。如果设置为 `False`，将显示扣除佣金前的交易结果。

## LogReturns

```python
class backtrader.observers.LogReturns()
```

该观察器存储策略的对数回报。

**参数**：

- `timeframe`（默认：无）：如果为 None，则报告整个回测期间的总回报。传递 `TimeFrame.NoTimeFrame` 以考虑整个数据集而不受时间限制。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 “TimeFrame.Minutes” 和 60 作为压缩在每小时时间框架上工作。
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。设置为 True 或 False 以获得特定行为。

记住，在运行的任何时刻都可以通过查看索引 0 处的线条名称来检查当前值。

## LogReturns2

```python
class backtrader.observers.LogReturns2()
```

扩展 `LogReturns` 观察器以显示两个工具。

## FundValue

```python
class backtrader.observers.FundValue(*args, **kwargs)
```

该观察器跟踪当前的基金价值。

**参数**：无

## FundShares

```python
class backtrader.observers.FundShares(*args, **kwargs)
```

该观察器跟踪当前的基金份额。

**参数**：无
