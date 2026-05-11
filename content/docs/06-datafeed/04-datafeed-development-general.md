---
title: "开发 Binary 数据源"
weight: 4
---

# 开发 Binary 数据源

**注意**：示例中使用的 Binary 文件 `goog.fd` 属于 VisualChart，不能与 backtrader 一起分发。

对于那些有兴趣直接使用 Binary 文件的人，可以免费下载 VisualChart。

CSV 数据源开发展示了如何添加基于 CSV 的数据源。基类 `CSVDataBase` 提供了框架，子类只需执行：

```python
def _loadline(self, linetokens):

  # 在这里解析 linetokens 并将它们放入 self.lines.close, self.lines.high 等中

  return True # 如果数据已解析，否则返回 False
```

基类负责参数、初始化、打开文件、读取行、标记化等事项，还会跳过不在日期范围（fromdate, todate）内的行。

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

- `dataname`：数据源识别如何获取数据的依据。在 CSVDataBase 中表示文件路径或类似文件的对象。
- `fromdate` 和 `todate`：定义传递给策略的日期范围，超出此范围的数据将被忽略。
- `name`：绘图时使用的名称。
- `timeframe`：时间框架参考。可选值：`Ticks`, `Seconds`, `Minutes`, `Days`, `Weeks`, `Months`, `Years`。
- `compression`（默认值：1）：每个实际条的压缩条数。仅在数据重采样/重放中有效。
- `sessionend`：如果传入 `datetime.time` 对象，将添加到数据源的日期时间行，用于标识会话结束。

## 示例二进制数据源

backtrader 已经定义了 VChartCSVData 用于 VisualChart 的导出数据，但也可以直接读取二进制数据文件。

下面来实现（完整代码见文末）。

### 初始化

二进制 VisualChart 数据文件包含每日数据（.fd 扩展名）或日内数据（.min 扩展名），通过参数 `timeframe` 区分文件类型。

在 `__init__` 中为每种类型设置不同的常量。

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

### 开始

数据源在回测开始时启动（优化期间可能启动多次）。

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

### 停止

回测结束时调用。

如果文件已打开，则将其关闭。

```python
def stop(self):
    # 如果有文件，关闭它
    if self.f is not None:
        self.f.close()
        self.f = None
```

### 实际加载

实际工作在 `_load` 中完成，用于加载下一组数据：datetime、open、high、low、close、volume、openinterest。在 backtrader 中，"当前"时刻对应于索引 0。

从文件中读取指定数量的字节（由 `__init__` 中的常量确定），使用 `struct` 模块解析，进行必要的处理（如日期时间的 divmod 操作），并存入数据源的行中。

如果无法读取数据，视为文件末尾（EOF），返回 `False`。

如果数据加载并解析成功，返回 `True`。

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

## 其他二进制格式

同样的模式适用于其他二进制源：

- 数据库
- 分层数据存储
- 在线来源

步骤总结：

- `__init__` -> 实例的初始化代码，只执行一次
- `start` -> 回测开始时执行（优化时将多次运行）
- `stop` -> 清理工作，如关闭数据库连接或套接字
- `_load` -> 从数据库或在线源获取下一组数据，加载到行对象中。标准字段：datetime、open、high、low、close、volume、openinterest

## VChartData 测试

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
