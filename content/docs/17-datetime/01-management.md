---
title: "管理"
weight: 1
---

# 管理

在 1.5.0 版本之前，backtrader 对时间管理采用直接方式，即使用数据源计算的日期时间。用户输入的参数（如 `fromdate` 或 `sessionstart`）也可传递给数据源。

这种方法在回测静态数据时效果很好，可以假设日期时间在进入系统前已处理好。

但 1.5.0 版本之后，backtrader 开始支持实时数据源，这就需要考虑日期时间管理。如果以下情况总是成立，则不需要这种管理：

- 纽约的交易者交易 ES-Mini。两者时区均为 US/Eastern。
- 柏林的交易者交易 DAX 期货。两者时区均为 CET（Europe/Berlin）。

上述直接方式可以工作，因为柏林的交易者可以这样写：

```python
class Strategy(bt.Strategy):

    def next(self):
        # DAX 期货在 CET 时间 08:00 开盘
        if self.data.datetime.time() < datetime.time(8, 30):
            # 市场运行 30 分钟前不操作
            return
```

但当同一个柏林交易者交易 ES-Mini 时，直接方式的问题就暴露了。由于夏令时（DST）转换时间不同（美国和欧洲的 DST 切换时间不一致），会导致一年中某些周的时间不同步。

以下代码并不总是有效：

```python
class Strategy(bt.Strategy):

    def next(self):
        # SPX 在 US/Eastern 时间全年 09:30 开盘
        # 对应 CET 大部分时间是 15:30
        # 但有时是 16:30 或 14:30，取决于美欧 DST 切换时间
        # 因此以下代码不可靠

        if self.data.datetime.time() < datetime.time(16, 0):
            # 市场运行 30 分钟前不操作
            return
```

## 使用时区操作

为解决上述问题并保持兼容性，backtrader 提供了以下选项：

### 日期时间输入

默认情况下，平台不会修改数据源提供的日期时间。

用户可以通过以下方式覆盖输入：

- 为数据源提供 `tzinput` 参数，该参数需兼容 `datetime.tzinfo` 接口，通常是一个 `pytz.timezone` 实例。

这样，backtrader 内部使用的时间被视为 UTC 类格式，即：

- 数据源本来就以 UTC 格式存储。
- 经过 `tzinput` 转换后。

它并非真正的 UTC，但因为是用户的参考系，所以称为 UTC 类。

### 日期时间输出

如果数据源能自动确定时区，默认使用自动确定的时区。

这在实时数据源场景下尤其有意义。例如，柏林（CET）的交易者交易 US/Eastern 的产品：

交易者始终看到正确的时间。在上例中，开盘时间在 US/Eastern 时区始终是 09:30，而不是一年中大部分时间的 15:30 CET，有时是 16:30 CET，有时是 14:30 CET。

如果不能确定，则输出为输入时确定的时间（UTC 类时间）。

用户可以通过以下方式覆盖输出时区：

- 为数据源提供 `tz` 参数，兼容 `datetime.tzinfo` 接口，通常是一个 `pytz.timezone` 实例。

注意：

用户输入的参数（如 `fromdate` 或 `sessionstart`）应与实际 `tz` 同步，无论 `tz` 是自动计算、用户提供还是默认（`None` 表示直接输入-输出）。

综合以上内容，来看柏林的交易者交易 US/Eastern 时区的产品：

```python
import pytz
import bt

data = bt.feeds.MyFeed('ES-Mini', tz=pytz.timezone('US/Eastern'))

class Strategy(bt.Strategy):

    def next(self):
        # 这将全年有效。
        # 数据源在 'US/Eastern' 时区框架内返回数据，
        # 以 '10:00' 作为参考时间。
        # 因为 SPX 在 US/Eastern 时区始终是 09:30 开盘。

        if self.data.datetime.time() < datetime.time(10, 0):
            # 市场运行 30 分钟前不操作
            return
```

对于可以自动确定输出时区的数据源：

```python
import bt

data = bt.feeds.MyFeedAutoTZ('ES-Mini')

class Strategy(bt.Strategy):

    def next(self):
        # 这将全年有效。
        # 数据源在 'US/Eastern' 时区框架内返回数据，
        # 以 '10:00' 作为参考时间。
        # 因为 SPX 在 US/Eastern 时区始终是 09:30 开盘。

        if self.data.datetime.time() < datetime.time(10, 0):
            # 市场运行 30 分钟前不操作
            return
```

上面示例中的 `MyFeed` 和 `MyFeedAuto` 仅为虚拟名称。

注意：

目前发行版中唯一能自动确定时区的数据源是连接到 Interactive Brokers 的数据源。
