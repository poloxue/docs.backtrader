---
title: "开发 CSV 数据源"
weight: 3
--- 

# CSV 数据源开发

**Backtrader** 已经提供了一些通用和特定的 CSV 数据源。

- GenericCSVData
- VisualChartCSVData
- YahooFinanceData（用于在线下载）
- YahooFinanceCSVData（用于已下载的数据）
- BacktraderCSVData（内部用于测试，但也可以使用）

即使如此，你可能仍需要开发对特定 CSV 数据源的支持。

实际上，框架的设计让这件事变得很简单。

## 步骤

1. 从 `backtrader.CSVDataBase` 继承
2. 根据需要定义任何参数
3. 在 `start` 方法中进行任何初始化
4. 在 `stop` 方法中进行任何清理
5. 定义一个 `_loadline` 方法，其中实际工作发生。此方法接收一个参数：`linetokens`。

顾名思义，这是根据分隔符（从基类继承）拆分当前行后得到的标记列表。

如果解析到新数据，则填充相应的行并返回 `True`。

如果没有数据可用（解析结束），则返回 `False`。框架会在没有更多行时自动处理。

框架已处理的事项：

- 打开文件（或接收类似文件的对象）
- 跳过标题行（如有）
- 读取行
- 标记化
- 预加载支持（一次性将整个数据加载到内存中）

下面使用 BacktraderCSVData 内部 CSV 解析代码的简化版本。这个版本不需要初始化或清理（比如打开和关闭套接字这类操作）。

注意：

backtrader 数据源需要填充以下行业标准字段：

- datetime
- open
- high
- low
- close
- volume
- openinterest

如果你的策略或数据浏览只需要收盘价，可以忽略其他字段（每次迭代会自动用 `float('NaN')` 填充）。

本例仅支持每日格式：

```python
import itertools
import backtrader as bt

class MyCSVData(bt.CSVDataBase):

    def start(self):
        # 对于此数据源类型无需做任何操作
        pass

    def stop(self):
        # 对于此数据源类型无需做任何操作
        pass

    def _loadline(self, linetokens):
        i = itertools.count(0)

        dttxt = linetokens[next(i)]
        # 格式为 YYYY-MM-DD
        y = int(dttxt[0:4])
        m = int(dttxt[5:7])
        d = int(dttxt[8:10])

        dt = datetime.datetime(y, m, d)
        dtnum = date2num(dt)

        self.lines.datetime[0] = dtnum
        self.lines.open[0] = float(linetokens[next(i)])
        self.lines.high[0] = float(linetokens[next(i)])
        self.lines.low[0] = float(linetokens[next(i)])
        self.lines.close[0] = float(linetokens[next(i)])
        self.lines.volume[0] = float(linetokens[next(i)])
        self.lines.openinterest[0] = float(linetokens[next(i)])

        return True
```

代码假设所有字段都存在且可转换为浮点数；日期时间采用固定 YYYY-MM-DD 格式，无需使用 `strptime` 解析。

通过添加处理空值和日期格式解析的代码，可以满足更复杂的需求。GenericCSVData 正是这样实现的。

## 警告

通过继承 GenericCSVData，可以支持很多格式。

让我们添加对 Sierra Chart 每日格式的支持（始终以 CSV 格式存储）。

定义（通过查看一个 '.dly' 数据文件）：

字段：Date, Open, High, Low, Close, Volume, OpenInterest

行业标准字段以及 GenericCSVData 已支持的字段，顺序相同（也是行业标准）

分隔符：,

日期格式：YYYY/MM/DD

一个用于这些文件的解析器：

```python
class SierraChartCSVData(backtrader.feeds.GenericCSVData):

    params = (('dtformat', '%Y/%m/%d'),)
```

这里只是重新定义了基类中的一个现有参数，只需要更改日期格式字符串。

Sierra Chart 的解析器就完成了。

下面是 GenericCSVData 的参数定义作为提醒：

```python
class GenericCSVData(feed.CSVDataBase):
    params = (
        ('nullvalue', float('NaN')),
        ('dtformat', '%Y-%m-%d %H:%M:%S'),
        ('tmformat', '%H:%M:%S'),

        ('datetime', 0),
        ('time', -1),
        ('open', 1),
        ('high', 2),
        ('low', 3),
        ('close', 4),
        ('volume', 5),
        ('openinterest', 6),
    )
```
