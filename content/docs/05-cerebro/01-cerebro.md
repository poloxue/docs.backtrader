---
title: "Cerebro"
weight: 1
---

# Cerebro

`Cerebro` 是 Backtrader 的核心类，负责整个系统的运行。

它的功能包括：

- 收集输入（数据源）、执行者（策略）、观察者、评论者（分析器）和记录者（编写器），确保系统的正常运行。
- 执行回测或实时数据供给和交易。
- 返回回测结果。
- 提供策略绘图功能。

## 创建 `Cerebro` 实例

创建 `Cerebro` 实例时，可以通过传递一些控制参数：

```python
cerebro = bt.Cerebro(**kwargs)
```

这些参数会影响系统的执行，具体的参数说明可以参考文档（也可以应用于后续的 `run` 方法）。

## 添加数据源

最常见的方式是使用 `cerebro.adddata(data)` 添加数据源，`data` 是已实例化的数据源。例如：

```python
data = bt.BacktraderCSVData(dataname='mypath.days', timeframe=bt.TimeFrame.Days)
cerebro.adddata(data)
```

### 数据的重采样与重放

`Cerebro` 也支持对数据进行重采样或重放：

**重采样**：

```python
data = bt.BacktraderCSVData(dataname='mypath.min', timeframe=bt.TimeFrame.Minutes)
cerebro.resampledata(data, timeframe=bt.TimeFrame.Days)
```

**重放数据**：

```python
data = bt.BacktraderCSVData(dataname='mypath.min', timeframe=bt.TimeFrame.Minutes)
cerebro.replaydata(data, timeframe=bt.TimeFrame.Days)
```

你可以同时使用多种类型的数据源，包括常规数据、重采样数据和重放数据。但需要确保它们的时间对齐。详见文档中的 **多时间框架** 和 **数据重采样** 部分。

## 添加策略

`Cerebro` 接受策略类并传递相关参数，即使没有优化，也可以使用以下方式添加策略：

```python
cerebro.addstrategy(MyStrategy, myparam1=value1, myparam2=value2)
```

### 策略优化

在优化时，参数需要作为可迭代对象传递。例如：

```python
cerebro.optstrategy(MyStrategy, myparam1=range(10, 20))
```

这会运行 `MyStrategy` 10次，`myparam1` 的值从 10 到 19。

## 添加其他组件

你可以通过以下方法为回测添加额外的功能：

- **`addwriter`**：记录回测数据。
- **`addanalyzer`**：分析回测结果。
- **`addobserver`** 或 **`addobservermulti`**：添加观察者，实时跟踪策略执行。

## 自定义经纪人

`Cerebro` 默认使用 Backtrader 内建的经纪人，但你也可以自定义经纪人：

```python
broker = MyBroker()
cerebro.broker = broker  # 通过 getbroker/setbroker 方法设置
```

## 接收通知

数据源或经纪人可能会发出通知。`Cerebro` 通过 `notify_store` 方法接收这些通知。处理通知有三种方式：

### 使用回调函数

你可以通过 `addnotifycallback(callback)` 向 `Cerebro` 添加回调函数，函数签名如下：

```python
callback(msg, *args, **kwargs)
```

### 在策略中覆盖 `notify_store` 方法

你也可以在策略类中直接覆盖 `notify_store` 方法来处理通知，签名如下：

```python
def notify_store(self, msg, *args, **kwargs):
    # 处理通知
```

### 子类化 `Cerebro`

通过子类化 `Cerebro` 并覆盖 `notify_store`，这是最不推荐的方法。

## 执行回测

回测执行通过以下方法进行：

```python
result = cerebro.run(**kwargs)
```

`run` 方法支持多种参数，可以在实例化时指定，详细说明可以参考文档。

## 标准观察者

`Cerebro` 默认会实例化三个标准观察者：

- 经纪人观察者：追踪现金和投资组合的价值。
- 交易观察者：记录每笔交易的效果。
- 买卖观察者：记录操作的执行时间。

如果不需要这些标准观察者，可以通过 `stdstats=False` 禁用它们。

## 返回回测结果

回测执行后，`Cerebro` 会返回策略实例，供你分析回测结果。你可以访问策略中的所有元素进行详细检查：

```python
result = cerebro.run(**kwargs)
```

### 优化时的返回结果

- 如果没有使用优化，`result` 将是一个策略实例列表。
- 如果进行了优化，`result` 将是一个列表的列表，每个内部列表对应一次优化运行后的策略实例。

**注意**：优化的默认行为已经更改，优化时只会返回分析器结果，优化的策略结果需要通过设置 `optreturn=False` 来获取。

## 提供绘图功能

如果你安装了 `matplotlib`，可以通过以下方式绘制回测图形：

```python
cerebro.plot()
```

## 回测流程概述

回测的执行逻辑大致如下：

1. 传递任何存储的通知。
2. 数据源提供下一组 tick/条。
   - 数据同步：多个数据源的数据会按时间对齐，确保不同时间框架的数据能够同时进行计算。
   - **版本更新**：1.9.0.99 版本引入了新的同步行为。
3. 通知策略有关订单、交易和现金的更新。
4. 经纪人接受排队订单，并根据新数据执行订单。
5. 调用策略的 `next` 方法，评估新数据并执行策略逻辑。
6. 更新观察者、指标、分析器等，触发其他活动。
7. 将数据写入目标（通过编写器）。

## 重要注意事项

在数据源传递新的一组数据时，这些数据是“已关闭”的。这意味着策略发出的订单将基于 **下一条数据** 执行，而不是当前数据。

