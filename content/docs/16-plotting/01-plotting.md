---
title: "Plotting"
weight: 1
---

# 绘图

回测虽然是基于计算的自动化过程，但可视化仍然非常重要。无论是已有算法的回测结果，还是数据产生的各种指标，绘图都能帮助你理解发生了什么，进而优化或产生新想法。

因此，backtrader 利用 matplotlib 内置了绘图功能，可以绘制数据馈送、指标、交易操作、现金流和投资组合价值的变化。

## 如何绘图

任何回测运行都可以通过调用一个方法进行绘图：

```python
cerebro.plot()
```

当然，这通常是最后一个命令。例如，以下简单代码使用了 backtrader 源代码中的一个示例数据：

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)

import backtrader as bt

class St(bt.Strategy):
    def __init__(self):
        self.sma = bt.indicators.SimpleMovingAverage(self.data)

data = bt.feeds.BacktraderCSVData(dataname='../../datas/2005-2006-day-001.txt')

cerebro = bt.Cerebro()
cerebro.adddata(data)
cerebro.addstrategy(St)
cerebro.run()
cerebro.plot()
```

这将生成以下图表：

![示例图表](image)

图表包含了 3 个观察器，由于缺乏任何交易，它们在这种情况下几乎没有意义：

1. **CashValue** 观察器：跟踪回测期间的现金和投资组合总价值（含现金）。
2. **Trade** 观察器：交易结束时显示实际盈亏。交易定义为开仓后仓位归零（包括多空转换）。
3. **BuySell** 观察器：在价格图上标记买入和卖出操作的位置。

这 3 个观察器由 cerebro 自动添加，可通过 `stdstats` 参数控制（默认：True）。如需禁用：

```python
cerebro = bt.Cerebro(stdstats=False)
```

或在运行时：

```python
cerebro = bt.Cerebro()
...
cerebro.run(stdstats=False)
```

## 绘图元素

除观察器外，以下 3 种元素也会被绘制：

1. 通过 `adddata`、`replaydata` 和 `resampledata` 添加到 Cerebro 的数据馈送。
2. 在策略中声明的指标（或通过 `addindicator` 添加到 cerebro 的指标，用于实验目的）。
3. 通过 `addobserver` 添加到 cerebro 的观察器。

观察器是与策略同步运行的线性对象，可访问整个生态系统，用于跟踪现金和价值等指标。

## 绘图选项

指标和观察器有几个选项可以控制它们在图表上的绘制方式，分为 3 大类：

1. 影响整个对象绘图行为的选项。
2. 影响单个线条绘图行为的选项。
3. 影响系统级别绘图选项的选项。

### 对象级绘图选项

这些选项由指标和观察器中的 `plotinfo` 数据集控制：

```python
plotinfo = dict(
    plot=True,
    subplot=True,
    plotname='',
    plotskip=False,
    plotabove=False,
    plotlinelabels=False,
    plotlinevalues=True,
    plotvaluetags=True,
    plotymargin=0.0,
    plotyhlines=[],
    plotyticks=[],
    plothlines=[],
    plotforce=False,
    plotmaster=None,
    plotylimited=True,
)
```

虽然 `plotinfo` 在类定义中显示为字典，但 backtrader 的元类机制会将其转换为可继承的对象，支持多重继承。例如：

如果子类将 `subplot=True` 改为 `subplot=False`，则后续子类会以 `False` 作为 `subplot` 的默认值。

有两种方法为这些参数赋值。让我们看一下 `SimpleMovingAverage` 实例化的第一种方法：

```python
sma = bt.indicators.SimpleMovingAverage(self.data, period=15, plotname='mysma')
```

从示例可知，`SimpleMovingAverage` 构造函数未使用的 **kwargs 会尝试解析为 `plotinfo` 值。`SimpleMovingAverage` 只定义了一个参数 `period`，因此 `plotname` 会匹配 `plotinfo` 中同名参数。

第二种方法：

```python
sma = bt.indicators.SimpleMovingAverage(self.data, period=15)
sma.plotinfo.plotname = 'mysma'
```

与 `SimpleMovingAverage` 一起实例化的 `plotinfo` 对象可以被访问，并且其中的参数也可以通过标准的 Python 点符号访问。这种语法更易于理解。

## 参数含义

- **plot**：是否绘制该对象。
- **subplot**：是否与数据一起绘制或在独立的子图中绘制。例如，移动平均线是绘制在数据上的，而随机指标和 RSI 则是在不同的刻度上绘制的。
- **plotname**：在图表上使用的名称，而不是类名。例如上面的 `mysma` 而不是 `SimpleMovingAverage`。
- **plotskip**（已弃用）：`plot` 的旧别名。
- **plotabove**：是否在数据之上绘制，否则在数据下方绘制。这仅在 `subplot=True` 时有效。
- **plotlinelabels**：是否在图表的图例中显示单个线条的名称，当 `subplot=False` 时。
- **plotlinevalues**：控制图例中是否包含指标和观察器线条的最后绘制值。可以通过每条线的 `_plotvalue` 控制。
- **plotvaluetags**：控制是否在线条的右侧绘制带有最后值的标签。可以通过每条线的 `_plotvaluetag` 控制。
- **plotymargin**：在图表上单个子图顶部和底部添加的边距。它是一个以 1 为基数的百分比。例如：0.05 表示 5%。
- **plothlines**：一个包含值的可迭代对象（在刻度内），在这些值处绘制水平线。例如，经典指标的超买、超卖区域通常在 70 和 30 处绘制线。
- **plotyticks**：一个包含值的可迭代对象（在刻度内），在这些值处特意放置刻度值。例如，强制刻度有 50 以识别刻度的中点。
- **plotyhlines**：一个包含值的可迭代对象（在刻度内），在这些值处绘制水平线。如果未定义上述任何项，那么水平线和刻度的位置将完全由此值控制。
- **plotforce**：如果所有其他方法都失败，这是一个最后的强制绘图机制。
- **plotmaster**：一个指标/观察器有一个主控，它是工作的数据。在某些情况下，可能希望与不同的主控一起绘图。
- **plotylimited**：目前仅适用于数据馈送。如果为 `True`（默认），其他线条在数据图上的绘图不会改变刻度。

## 线条特定绘图选项

指标/观察器有线条，如何绘制这些线条可以通过 `plotlines` 对象进行影响。大多数 `plotlines` 中的选项旨在直接传递给 matplotlib。文档主要通过示例进行说明。

重要：选项是按每条线条指定的。

一些选项由 backtrader 直接控制，这些选项都以 `_` 开头：

- **_plotskip**（布尔值）：如果设置为 `True`，表示跳过绘制特定线条。
- **_plotvalue**（布尔值）：控制此线条的图例是否包含最后绘制的值（默认值为 `True`）。
- **_plotvaluetag**（布尔值）：控制是否在线条右侧绘制带有最后值的标签（默认值为 `True`）。
- **_name**（字符串）：更改特定线条的绘图名称。
- **_skipnan**（布尔值，默认：`False`）：跳过绘制 NaN 值。
- **_samecolor**（布尔值）：强制下一条线条使用与前一条相同的颜色，避免 matplotlib 为每个新绘制的元素循环颜色图。
- **_method**（字符串）：选择 matplotlib 用于绘制元素的方法。

示例：

- **MACDHisto** 使用 `_method='bar'` 绘制直方图。

- **BuySell** 观察器：
```python
plotlines = dict(
    buy=dict(marker='^', markersize=8.0, color='lime', fillstyle='full'),
    sell=dict(marker='v', markersize=8.0, color='red', fillstyle='full')
)
```

- **Trades** 观察器：
```python
lines = ('pnlplus', 'pnlminus')
plotlines = dict(
    pnlplus=dict(_name='Positive', marker='o', color='blue', markersize=8.0, fillstyle='full'),
    pnlminus=dict(_name='Negative', marker='o', color='red', markersize=8.0, fillstyle='full')
)
```

## 系统级绘图选项

cerebro 的 `plot` 方法签名：
```python
def plot(self, plotter=None, numfigs=1, iplot=True, **kwargs):
```

- **plotter**：一个包含控制系统级绘图选项的对象/类。如果为 `None`，则创建默认 `PlotScheme` 对象。
- **numfigs**：将图表分解为多少个独立的图表。
- **iplot**：如果在 Jupyter Notebook 中运行，自动内联绘图。
- **kwargs**：将用于更改 `plotter` 或默认 `PlotScheme` 对象的属性值。

## PlotScheme

包含控制系统级绘图的所有选项的对象。选项在代码中有文档说明。

### 颜色

`PlotScheme` 类定义了一个方法，可以在子类中重写，该方法返回要使用的下一个颜色：

```python
def color(self, idx)
```

其中 `idx` 是当前线条的索引。默认颜色方案是 Tableau 10 颜色调色板，索引修改为：

```python
tab10_index = [3, 0, 2, 1, 2, 4, 5, 6, 7, 8, 9]
```
