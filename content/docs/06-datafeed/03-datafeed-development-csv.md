---
title: "开发 CSV 数据源"
weight: 3
--- 

# CSV 数据源开发

**Backtrader** 已经提供了一些通用 CSV 数据源和特定的 CSV 数据源。

- GenericCSVData
- VisualChartCSVData
- YahooFinanceData（用于在线下载）
- YahooFinanceCSVData（用于已下载的数据）
- BacktraderCSVData（内部使用...用于测试目的，但也可以使用）

即使如此，最终用户可能仍希望开发对特定 CSV 数据源的支持。

通常的格言是：“说起来容易做起来难”。实际上，结构旨在使其变得简单。

## 步骤

1. 从 `backtrader.CSVDataBase` 继承
2. 根据需要定义任何参数
3. 在 `start` 方法中进行任何初始化
4. 在 `stop` 方法中进行任何清理
5. 定义一个 `_loadline` 方法，其中实际工作发生。此方法接收一个参数：`linetokens`。

顾名思义，这包含根据分隔符参数（从基类继承）拆分当前行后的标记。

如果在完成其工作后有新数据……填充相应的行并返回 `True`。

如果没有可用的数据，因此解析已结束：返回 `False`。

如果后台代码发现没有更多行需要解析，则可能不需要返回 `False`。

已考虑的事项：

- 打开文件（或接收类似文件的对象）
- 跳过标头行（如果指示存在）
- 读取行
- 标记行
- 预加载支持（将整个数据源一次性加载到内存中）

通常一个示例胜过千言万语。让我们使用 BacktraderCSVData 中定义的内部 CSV 解析代码的简化版本。这个版本不需要初始化或清理（例如，这可能是打开一个套接字并稍后关闭它）。

注意：

backtrader 数据源包含通常的行业标准源，这些源是要填充的。即：

- datetime
- open
- high
- low
- close
- volume
- openinterest

如果您的策略/算法或简单数据浏览只需要，例如收盘价，您可以不触碰其他字段（每次迭代会自动用 `float('NaN')` 值填充它们，然后用户代码有机会进行任何操作）。

在此示例中，仅支持每日格式：

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

代码假设所有字段都到位且可转换为浮点数，除了日期时间，它具有固定的 YYYY-MM-DD 格式，可以不使用 `datetime.datetime.strptime` 进行解析。

通过添加一些代码行来处理空值和日期格式解析，可以满足更复杂的需求。GenericCSVData 就是这样做的。

## 警告

使用现有的 GenericCSVData 和继承，可以实现很多格式支持。

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

参数定义只是重新定义了基类中的一个现有参数。在这种情况下，只需要更改日期格式字符串。

## 完成

Sierra Chart 的解析器已经完成。

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
