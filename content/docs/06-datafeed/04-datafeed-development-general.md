---
title: "开发二进制数据源"
weight: 4
---

### 二进制数据源开发

**注意**

示例中使用的二进制文件 `goog.fd` 属于 VisualChart，不能与 backtrader 一起分发。

对于那些有兴趣直接使用二进制文件的人，可以免费下载 VisualChart。

CSV 数据源开发展示了如何添加新的基于 CSV 的数据源。现有的基类 `CSVDataBase` 提供了框架，减轻了子类的大部分工作，在大多数情况下，它们可以简单地执行：

```python
def _loadline(self, linetokens):

  # 在这里解析 linetokens 并将它们放入 self.lines.close,
  # self.lines.high 等中

  return True # 如果数据已解析，否则返回 False
```

基类负责参数、初始化、打开文件、读取行、将行拆分为标记以及其他事项，例如跳过不符合日期范围（fromdate，todate）的行，这些行可能由最终用户定义。

开发非 CSV 数据源遵循相同的模式，而无需深入到已拆分的行标记。

### 需要做的事情：

1. 从 `backtrader.feed.DataBase` 派生
2. 添加任何需要的参数
3. 如果需要初始化，重写 `__init__(self)` 和/或 `start(self)`
4. 如果需要清理代码，重写 `stop(self)`
5. 工作发生在必须始终重写的方法 `_load(self)` 内

让我们看看 `backtrader.feed.DataBase` 已经提供的参数：

```python
from backtrader.utils.py3 import with_metaclass

...
...

class DataBase(with_metaclass(MetaDataBase, dataseries.OHLCDateTime)):

    params = (('dataname', None),
        ('fromdate', datetime.datetime.min),
        ('todate', datetime.datetime.max),
        ('name', ''),
        ('compression', 1),
        ('timeframe', TimeFrame.Days),
        ('sessionend', None))
```

这些参数具有以下含义：

- `dataname`：允许数据源识别如何获取数据。在 CSVDataBase 的情况下，此参数表示文件路径或已存在的类似文件的对象。
- `fromdate` 和 `todate`：定义传递给策略的日期范围。数据源提供的任何超出此范围的值都将被忽略。
- `name`：用于绘图目的的装饰名称。
- `timeframe`：表示时间工作参考。潜在值：`Ticks`, `Seconds`, `Minutes`, `Days`, `Weeks`, `Months` 和 `Years`。
- `compression`（默认值：1）：每条实际条的条数。信息性。仅在数据重采样/重放中有效。
- `sessionend`：如果传递（`datetime.time` 对象），将添加到数据源日期时间行，允许识别会话结束。

### 示例二进制数据源

backtrader 已经定义了一个 CSV 数据源（VChartCSVData）用于 VisualChart 的导出数据，但也可以直接读取二进制数据文件。

让我们来实现（完整的数据源代码可以在文末找到）。

#### 初始化

二进制 VisualChart 数据文件可以包含每日数据（.fd 扩展名）或日内数据（.min 扩展名）。这里使用参数 `timeframe` 来区分读取的文件类型。

在 `__init__` 中，设置每种类型不同的常量。

```python
def __init__(self):
    super(VChartData, self).__init__()

    # 使用 informative "timeframe" 参数来理解传递的 "dataname"
    # 是指日内还是每日数据源
    if self.p.timeframe >= TimeFrame.Days:
        self.barsize = 28
        self.dtsize = 1
        self.barfmt = 'IffffII'
    else:
        self.dtsize = 2
        self.barsize = 32
        self.barfmt = 'IIffffII'
```

#### 开始

数据源将在回测开始时启动（在优化期间实际上可以启动多次）。

在 `start` 方法中，除非传递了类似文件的对象，否则二进制文件会被打开。

```python
def start(self):
    # 数据源必须启动...打开文件（或查看是否已打开）
    self.f = None
    if hasattr(self.p.dataname, 'read'):
        # 传入了文件（例如：来自 GUI）
        self.f = self.p.dataname
    else:
        # 让异常传播
        self.f = open(self.p.dataname, 'rb')
```

#### 停止

回测结束时调用。

如果文件已打开，则将其关闭。

```python
def stop(self):
    # 如果有文件，关闭它
    if self.f is not None:
        self.f.close()
        self.f = None
```

#### 实际加载

实际工作在 `_load` 中完成。调用以加载下一组数据，在这种情况下是下一个：datetime、open、high、low、close、volume、openinterest。在 backtrader 中，“实际”时刻对应于索引 0。

将从打开的文件中读取一些字节（由 `__init__` 中设置的常量确定），使用 `struct` 模块解析，如果需要进一步处理（如日期和时间的 divmod 操作），并存储在数据源的行中：datetime、open、high、low、close、volume、openinterest。

如果无法从文件中读取数据，则假定已达到文件末尾（EOF）

返回 `False` 以指示没有更多数据可用

如果数据已加载并解析：

返回 `True` 以指示数据集加载成功

```python
def _load(self):
    if self.f is None:
        # 如果没有文件...无法解析
        return False

    # 读取所需数量的二进制数据
    bardata = self.f.read(self.barsize)
    if not bardata:
        # 如果没有读取数据...游戏结束返回 "False"
        return False

    # 使用 struct 解析数据
    bdata = struct.unpack(self.barfmt, bardata)

    # 年份存储为每年 500 天
    y, md = divmod(bdata[0], 500)
    # 月份存储为每月 32 天
    m, d = divmod(md, 32)
    # 将 y, m, d 放入 datetime
    dt = datetime.datetime(y, m, d)

    if self.dtsize > 1:  # 分钟条
        # 每日时间以秒为单位存储
        hhmm, ss = divmod(bdata[1], 60)
        hh, mm = divmod(hhmm, 60)
        # 将时间添加到现有的 datetime
        dt = dt.replace(hour=hh, minute=mm, second=ss)

    self.lines.datetime[0] = date2num(dt)

    # 获取解析的数据的其余部分
    o, h, l, c, v, oi = bdata[self.dtsize:]
    self.lines.open[0] = o
    self.lines.high[0] = h
    self.lines.low[0] = l
    self.lines.close[0] = c
    self.lines.volume[0] = v
    self.lines.openinterest[0] = oi

    # 返回成功
    return True
```

### 其他二进制格式

同样的模型可以应用于任何其他二进制源：

- 数据库
- 分层数据存储
- 在线来源

步骤再次说明：

- `__init__` -> 实例的任何初始化代码，仅一次
- `start` -> 回测开始时（如果将进行优化则会多次运行）
- `stop` -> 清理，例如关闭数据库连接或打开的套接字
- `_load` -> 查询数据库或在线源以获取下一组数据并将其加载到对象的行中。标准字段为：datetime、open、high、low、close、volume、openinterest

### VChartData 测试

VChartData 从本地 `.fd` 文件加载 2006 年 Google 的数据。

仅涉及加载数据，因此不需要策略的子类。

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import datetime

import backtrader as bt
from vchart import VChartData


if __name__ == '__main__':
    # 创建 cerebro 实体
    cerebro = bt.Cerebro(stdstats=False)

    # 添加策略
    cerebro.addstrategy(bt.Strategy)

    ###########################################################################
    # 注意：
    # goog.fd 文件属于 VisualChart，不能与 backtrader 一起分发
    #
    # VisualChart 可从 www.visualchart.com 下载
    ###########################################################################
    # 创建数据源
    datapath = '../../datas/goog.fd'
    data = VChartData(
        dataname=datapath,
        fromdate=datetime.datetime(2006, 1, 1),
        todate=datetime.datetime(2006, 12, 31),
        timeframe=bt.TimeFrame.Days
    )

    # 将数据源添加到 Cerebro
    cerebro.adddata(data)

    # 运行所有内容
    cerebro.run()

    # 绘制结果
    cerebro.plot(style='bar')
```

### VChartData 完整代码

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import datetime
import struct

from backtrader.feed import DataBase
from backtrader import date2num
from backtrader import TimeFrame


class VChartData(DataBase):
    def __init__(self):
        super(VChartData, self).__init__()

        # 使用 informative "timeframe" 参数来理解传递的 "dataname"
        # 是指日内还是每日数据源
        if self.p.timeframe >= TimeFrame.Days:
            self.barsize = 28
            self.dtsize = 1
            self.barfmt = 'IffffII'
        else:
            self.dtsize = 2
            self.barsize = 32
            self.barfmt = 'IIffffII'

    def start(self):
        # 数据源必须启动...打开文件（或查看是否已打开）
        self.f = None
        if hasattr(self.p.dataname, 'read'):
            # 传入了文件（例如：来自 GUI）
            self.f = self.p.dataname
        else:
            # 让异常传播
            self.f = open(self.p.dataname, 'rb')

    def stop(self):
        # 如果有文件，关闭它
        if self.f is not None:
            self.f.close()
            self.f = None

    def _load(self):
        if self.f is None:
            # 如果没有文件...无法解析
            return False

        # 读取所需数量的二进制数据
        bardata = self.f.read(self.barsize)
        if not bardata:
            # 如果没有读取数据...游戏结束返回 "False"
            return False

        # 使用 struct 解析数据
        bdata = struct.unpack(self.barfmt, bardata)

        # 年份存储为每年 500 天
        y, md = divmod(bdata[0], 500)
        # 月份存储为每月 32 天
        m, d = divmod(md, 32)
        # 将 y, m, d 放入 datetime
        dt = datetime.datetime(y, m, d)

        if self.dtsize > 1:  # 分钟条
            # 每日时间以秒为单位存储
            hhmm, ss = divmod(bdata[1], 60)
            hh, mm = divmod(hhmm, 60)
            # 将时间添加到现有的 datetime
            dt = dt.replace(hour=hh, minute=mm, second=ss)

        self.lines.datetime[0] = date2num(dt)

        # 获取解析的数据的其余部分
        o, h, l, c, v, oi = bdata[self.dtsize:]
        self.lines.open[0] = o
        self.lines.high[0] = h
        self.lines.low[0] = l
        self.lines.close[0] = c
        self.lines.volume[0] = v
        self.lines.openinterest[0] = oi

        # 返回成功
        return True
```
