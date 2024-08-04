---
title: "Data Feeds"
weight: 1
---

### 数据源

backtrader 提供了一组数据源解析器（在撰写本文时都是基于 CSV 的）以便从不同来源加载数据。

- Yahoo（在线或已保存到文件）
- VisualChart（参见 www.visualchart.com）
- Backtrader CSV（自定义格式用于测试）
- 通用 CSV 支持

从快速入门指南中可以清楚地看到，您可以将数据源添加到 Cerebro 实例中。这些数据源稍后将在策略中可用：

- `self.datas` 数组（按插入顺序）
- 数组对象的别名：
  - `self.data` 和 `self.data0` 指向第一个元素
  - `self.dataX` 指向数组中索引为 X 的元素

以下是插入方式的快速提醒：

```python
import backtrader as bt
import backtrader.feeds as btfeeds

data = btfeeds.YahooFinanceCSVData(dataname='wheremydatacsvis.csv')

cerebro = bt.Cerebro()
cerebro.adddata(data)  # 可以传递一个 'name' 参数用于绘图
```

### 数据源通用参数

这个数据源可以直接从 Yahoo 下载数据并将其输入系统。

参数：

- `dataname`（默认：None）必须提供
  - 含义随数据源类型而异（文件位置、股票代码等）

- `name`（默认：''）
  - 用于绘图装饰目的。如果未指定，可能会从 `dataname` 派生（例如：文件路径的最后一部分）

- `fromdate`（默认：`mindate`）
  - Python datetime 对象，表示应忽略此日期之前的任何日期时间

- `todate`（默认：`maxdate`）
  - Python datetime 对象，表示应忽略此日期之后的任何日期时间

- `timeframe`（默认：`TimeFrame.Days`）
  - 可能的值：Ticks、Seconds、Minutes、Days、Weeks、Months 和 Years

- `compression`（默认：1）
  - 每个实际条形图的条形数。信息性。仅在数据重采样/重放中有效。

- `sessionstart`（默认：None）
  - 数据的会话开始时间指示。可由类用于重采样等目的

- `sessionend`（默认：None）
  - 数据的会话结束时间指示。可由类用于重采样等目的

### CSV 数据源通用参数

参数（除了通用参数外）：

- `headers`（默认：True）
  - 指示传递的数据是否具有初始标题行

- `separator`（默认：“，”）
  - 分隔符，用于标记 CSV 每行中的各个字段

### `GenericCSVData`

此类公开了一个通用接口，允许解析几乎所有的 CSV 文件格式。

根据参数定义的顺序和字段存在情况解析 CSV 文件。

特定参数（或特定含义）：

- `dataname`
  - 要解析的文件名或类似文件的对象

- `datetime`（默认：0）
  - 包含日期（或日期时间）字段的列

- `time`（默认：-1）
  - 如果与日期时间字段分开，则包含时间字段的列（-1 表示不存在）

- `open`（默认：1）、`high`（默认：2）、`low`（默认：3）、`close`（默认：4）、`volume`（默认：5）、`openinterest`（默认：6）
  - 包含相应字段的列的索引
  - 如果传递负值（例如：-1），则表示 CSV 数据中不存在该字段

- `nullvalue`（默认：float('NaN')）
  - 如果缺少应有的值（CSV 字段为空），将使用的值

- `dtformat`（默认：`%Y-%m-%d %H:%M:%S`）
  - 用于解析日期时间 CSV 字段的格式

- `tmformat`（默认：`%H:%M:%S`）
  - 如果存在，用于解析时间 CSV 字段的格式（默认情况下时间 CSV 字段不存在）

### 示例使用

满足以下要求的示例用法：

- 限制输入年份为 2000
- HLOC 顺序而不是 OHLC
- 将缺失值替换为零（0.0）
- 提供日线数据，日期时间只是格式为 YYYY-MM-DD 的日期
- 没有 `openinterest` 列

代码如下：

```python
import datetime
import backtrader as bt
import backtrader.feeds as btfeeds

...
...

data = btfeeds.GenericCSVData(
    dataname='mydata.csv',

    fromdate=datetime.datetime(2000, 1, 1),
    todate=datetime.datetime(2000, 12, 31),

    nullvalue=0.0,

    dtformat=('%Y-%m-%d'),

    datetime=0,
    high=1,
    low=2,
    open=3,
    close=4,
    volume=5,
    openinterest=-1
)
...
```

略微修改后的要求：

- 限制输入年份为 2000
- HLOC 顺序而不是 OHLC
- 将缺失值替换为零（0.0）
- 提供日内数据，带有单独的日期和时间列
- 日期格式为 YYYY-MM-DD
- 时间格式为 HH.MM.SS（而不是通常的 HH:MM:SS）
- 没有 `openinterest` 列

代码如下：

```python
import datetime
import backtrader as bt
import backtrader.feeds as btfeeds

...
...

data = btfeeds.GenericCSVData(
    dataname='mydata.csv',

    fromdate=datetime.datetime(2000, 1, 1),
    todate=datetime.datetime(2000, 12, 31),

    nullvalue=0.0,

    dtformat=('%Y-%m-%d'),
    tmformat=('%H.%M.%S'),

    datetime=0,
    time=1,
    high=2,
    low=3,
    open=4,
    close=5,
    volume=6,
    openinterest=-1
)
```

这也可以通过子类化永久保存：

```python
import datetime
import backtrader.feeds as btfeeds

class MyHLOC(btfeeds.GenericCSVData):

  params = (
    ('fromdate', datetime.datetime(2000, 1, 1)),
    ('todate', datetime.datetime(2000, 12, 31)),
    ('nullvalue', 0.0),
    ('dtformat', ('%Y-%m-%d')),
    ('tmformat', ('%H.%M.%S')),

    ('datetime', 0),
    ('time', 1),
    ('high', 2),
    ('low', 3),
    ('open', 4),
    ('close', 5),
    ('volume', 6),
    ('openinterest', -1)
)
```

现在可以通过提供 `dataname` 重用此新类：

```python
data = btfeeds.MyHLOC(dataname='mydata.csv')
```
