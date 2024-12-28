---
title: "启动运行"
weight: 2
---

# 启动和运行

`Backtrader` 的启动和运行过程主要依赖于三个核心组件：

1. **数据源**：提供市场数据，用于回测或实时交易。
2. **策略**：定义交易逻辑（基于类继承实现）。
3. **Cerebro**：核心管理器，负责整合数据源、策略，并启动回测或实时交易。

---

## **数据源**

数据源是回测和策略运行的基础，它为策略提供价格数据（如开盘价、高价、低价、收盘价）及其他市场信息。

### **支持的数据源**

**本地数据文件**：

- 支持多种 CSV 格式（如 Yahoo Finance 数据）。
- 支持从 Pandas DataFrame 加载数据。

**在线数据提取**：

- 提供内置的 Yahoo Finance 在线数据提取功能。

**实时数据源**：

- 支持 Interactive Brokers (IB)、Visual Chart 和 Oanda 等实时数据源。

平台支持通过 **时间框架**（如日线、5分钟线）和 **压缩级别**（如1天、5分钟）自定义数据，以适配不同交易策略。

---

### **数据源设置示例**

#### **加载 Yahoo Finance 格式的 CSV 数据**

以下是一个基本的 CSV 数据加载示例：

```python
import backtrader as bt
import datetime

datapath = 'path/to/your/yahoo/data.csv'

data = bt.feeds.YahooFinanceCSVData(
    dataname=datapath,
    reversed=True  # 如果数据是从最新日期到最早日期排列，需要设置为 True
)
```

如果数据跨越时间范围较长，可限制按时间限制加载的数据：


```python
data = bt.feeds.YahooFinanceCSVData(
    dataname=datapath,
    reversed=True,  # 如果数据是从最新日期到最早日期排列，需要设置为 True
    fromdate=datetime.datetime(2014, 1, 1),  # 数据起始日期
    todate=datetime.datetime(2014, 12, 31),  # 数据结束日期
    timeframe=bt.TimeFrame.Days,  # 时间框架设置为日线
    compression=1,  # 每 1 天作为一个数据单位
    name='Yahoo Data'  # 数据源命名（可选）
)
```

#### **从 Pandas DataFrame 加载数据**
如果你的数据存储在 Pandas DataFrame 中，可以使用以下方式加载：

```python
import pandas as pd
import backtrader as bt

df = pd.read_csv('path/to/your/data.csv', index_col='Date', parse_dates=True)

data = bt.feeds.PandasData(
    dataname=df,
    fromdate=datetime.datetime(2020, 1, 1),
    todate=datetime.datetime(2020, 12, 31),
    timeframe=bt.TimeFrame.Days
)
```


### **设置多时间框架**

如果你需要同时加载 5 分钟和日线数据，可以分别加载不同时间框架的数据源：

```python
data_5min = bt.feeds.GenericCSVData(
    dataname='path/to/5min_data.csv',
    timeframe=bt.TimeFrame.Minutes,
    compression=5
)

data_daily = bt.feeds.GenericCSVData(
    dataname='path/to/daily_data.csv',
    timeframe=bt.TimeFrame.Days,
    compression=1
)

cerebro.adddata(data_5min, name='5min')
cerebro.adddata(data_daily, name='daily')
```

---

## **策略（派生类）**

**策略** 是交易逻辑的核心，它定义了如何根据数据进行买卖操作。Backtrader 中的策略通过继承 `bt.Strategy` 类实现。

### **策略的基本方法**

策略类至少需要实现以下两个方法：

1. **`__init__`**：初始化方法，用于定义指标和变量。
2. **`next`**：逐条处理数据，用于执行交易逻辑。

当传递不同时间框架（条数不同）的数据源时，next 方法将基于主数据源迭代（即第一个传递给 Cerebro 的数据），主数据源应是时间框架最小的数据源。

对于 `ReplayData`，`next` 方法会被多次调用，特别是当数据跨时间框架时，**Backtrader** 会在每个数据条上执行 next，模拟数据的发展。

### **示例策略**

以下是一个简单的策略示例，基于 20 日简单移动平均线（SMA）进行交易：

```python
class MyStrategy(bt.Strategy):
    def __init__(self):
        # 定义简单移动平均线（SMA）
        self.sma = bt.indicators.SimpleMovingAverage(self.data.close, period=20)

    def next(self):
        # 如果收盘价高于 SMA，则买入
        if self.data.close[0] > self.sma[0]:
            self.buy()
        # 如果收盘价低于 SMA，则卖出
        elif self.data.close[0] < self.sma[0]:
            self.sell()
```

---

### **扩展功能**

策略还可以覆盖以下方法，为回测和实时交易添加更多功能：

1. **`start`**：回测开始时调用（用于初始化资源）。
2. **`stop`**：回测结束时调用（用于清理资源或总结）。
3. **`notify_order`**：当订单状态发生变化时调用。
4. **`notify_trade`**：当交易状态发生变化时调用。

演示策略如下所示：

```python
class MyStrategy(bt.Strategy):
    def __init__(self):
        self.sma = bt.indicators.SimpleMovingAverage(self.data.close, period=20)

    def start(self):
        print('回测开始！')

    def stop(self):
        print('回测结束！')

    def notify_order(self, order):
        if order.status in [order.Completed]:
            print(f"订单完成: {order.info}")

    def next(self):
        if self.data.close[0] > self.sma[0]:
            self.buy()
        elif self.data.close[0] < self.sma[0]:
            self.sell()
```

### 其他操作

在 Backtrader 中，策略类还提供各种交易操作。

- **买入 / 卖出 / 平仓（buy / sell / close）**：
  通过这些方法，策略可以向经纪人发送买入、卖出或平仓订单。平台也允许您手动创建订单并将其传递给经纪人，但通过这些内建的方法，可以让操作变得更简单和高效。

- **close**：立即关闭当前市场头寸。

- **getposition**（或属性 `position`）：返回当前的市场头寸。通过这个方法，您可以查看当前持有的仓位情况。

- **setsizer / getsizer**（或属性 `sizer`）：这些方法允许您设置或获取底层的股份定量器（Sizer）。定量器负责计算每次交易的仓位大小，您可以根据需要选择不同的定量器类型，如固定大小、与资本成比例或指数等策略。

策略类本身是一个 `Line` 对象，支持配置参数。

```python
class MyStrategy(bt.Strategy):

    params = (('period', 20),)

    def __init__(self):
        self.sma = btind.SimpleMovingAverage(self.data, period=self.params.period)
    ...
```

现在，`SimpleMovingAverage` 指标不再是固定的 `20`，而是根据策略中定义的 `period` 参数动态设置，这提高了策略的灵活性。

---

## **Cerebro**

一旦数据源可用并定义了策略，Cerebro 实例会将所有内容整合并执行。创建一个 `Cerebro` 实例非常简单：

```python
cerebro = bt.Cerebro()
```

如果没有特别的需求，默认配置会自动处理以下内容：

- 创建一个默认经纪人；
- 操作不收取佣金；
- 数据源会被预加载；
- 默认执行模式为 `runonce`（批处理模式），这是最优的执行方式。

需要注意的是，所有指标都必须支持 `runonce` 模式，以确保执行速度达到最佳。平台中已经包含的指标大多都支持此模式。

### **典型流程**

**创建 `Cerebro` 实例**

```python
cerebro = bt.Cerebro()
```

**添加数据源**

```python
cerebro.adddata(data)
```

**添加策略**
```python
cerebro.addstrategy(MyStrategy)
```

**配置经纪人**

设置初始资金、佣金等。

```python
cerebro.broker.set_cash(100000)  # 设置初始资金
cerebro.broker.setcommission(commission=0.001)  # 设置佣金
```

**运行回测**

```python
cerebro.run()
```

**绘制回测结果**

```python
cerebro.plot()
```

plot接受一些参数用于自定义：

- numfigs=1

如果绘图过于密集，可以将其分成多个图

- plotter=None

可以传递一个绘图器实例给，cerebro 将不按默认绘图。

---

### **策略优化**

Backtrader 支持对策略参数进行优化。例如，我们可以测试不同的 SMA 周期参数对策略的影响：

```python
cerebro.optstrategy(MyStrategy, period=range(10, 50, 5))
```

在上例中，`period` 参数会从 10 到 45（每次递增 5）进行测试，平台会自动运行每个参数组合的回测。

---

## **完整代码示例**

以下是一个完整的代码示例，整合了数据源、策略和 Cerebro：

```python
import backtrader as bt
from datetime import datetime

# 定义策略
class MyStrategy(bt.Strategy):
    def __init__(self):
        self.sma = bt.indicators.SimpleMovingAverage(self.data.close, period=20)

    def next(self):
        if self.data.close[0] > self.sma[0]:
            self.buy()
        elif self.data.close[0] < self.sma[0]:
            self.sell()

# 初始化 Cerebro
cerebro = bt.Cerebro()

# 加载数据
data = bt.feeds.YahooFinanceCSVData(
    dataname='path/to/your/data.csv',
    fromdate=datetime(2020, 1, 1),
    todate=datetime(2021, 1, 1)
)
cerebro.adddata(data)

# 添加策略
cerebro.addstrategy(MyStrategy)

# 配置初始资金和佣金
cerebro.broker.setcash(100000)
cerebro.broker.setcommission(commission=0.001)

# 启动回测
cerebro.run()

# 绘制结果
cerebro.plot()
```

---

通过这些核心组件的整合，**Backtrader** 提供了一个高度灵活的框架，适用于各种回测和实时交易需求。希望这部分的补充对你有所帮助！
