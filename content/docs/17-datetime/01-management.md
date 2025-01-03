---
title: "管理"
weight: 1
---

# 管理

在发布 1.5.0 版本之前，backtrader 对时间管理采用的是直接方式，即直接使用数据源计算出的任何日期时间。用户输入的参数，如 `fromdate`（或 `sessionstart`），也可以传递给任何数据源。

这种方法在对冻结数据源进行回测时效果很好。可以假设输入的日期时间在进入系统之前已经经过处理。

但在 1.5.0 版本之后，backtrader 开始支持实时数据源，这就需要考虑日期时间管理。如果以下情况总是成立，那么就不需要进行这种管理：

- 纽约的交易者交易 ES-Mini。这两个的时区都是 US/Eastern（或其别名）。
- 柏林的交易者交易 DAX 期货。在这种情况下，两个的时区都是 CET（或 Europe/Berlin）。

上面的直接输入-输出日期时间方法可以工作，因为柏林的交易者可以始终这样做：

```python
class Strategy(bt.Strategy):

    def next(self):
        # DAX 期货在 CET 时间早上 08:00 开盘
        if self.data.datetime.time() < datetime.time(8, 30):
            # 市场运行 30 分钟之前不操作
            return  #
```

当同一个柏林交易者决定交易 ES-Mini 时，直接方法的问题就会显现出来。因为 DST（夏令时）的变化发生在一年中的不同时间，这会导致时间差异在一年中的某些周内不同步。

以下代码并不总是有效：

```python
class Strategy(bt.Strategy):

    def next(self):
        # SPX 在 US/Eastern 全年早上 09:30 开盘
        # 大部分时间是 15:30 CET
        # 但有时是 16:30 CET 或 14:30 CET，取决于美国和欧洲的 DST 切换时间
        # 因此以下代码是不可靠的

        if self.data.datetime.time() < datetime.time(16, 0):
            # 市场运行 30 分钟之前不操作
            return  #
```

## 使用时区操作

为了解决上述问题并仍然保持与直接输入-输出时间方法的兼容性，backtrader 为终端用户提供了以下选项：

### 日期时间输入

默认情况下，平台不会触及数据源提供的日期时间。

用户可以通过以下方式覆盖此输入：

- 为数据源提供 `tzinput` 参数。这必须是与 `datetime.tzinfo` 接口兼容的对象。用户很可能会提供一个 `pytz.timezone` 实例。

通过这个决定，backtrader 内部使用的时间被认为是 UTC 类格式，即：

- 如果数据源已经以 UTC 格式存储它。
- 通过 `tzinput` 转换后。

它实际上不是 UTC，但它是用户的参考，因此是 UTC 类的。

### 日期时间输出

如果数据源可以自动确定时区，这将是默认设置。

这在实时数据源的情况下尤其有意义，例如柏林（CET 时区）的交易者交易 US/Eastern 时区的产品。

因为交易者总是得到正确的时间，在上面的示例中，开盘时间在 US/Eastern 时区内始终是早上 09:30，而不是一年中大部分时间的 15:30 CET，有时是 16:30 CET，有时是 14:30 CET。

如果不能确定，那么输出将是输入时确定的时间（UTC 类时间）。

用户可以覆盖并确定输出的实际时区：

- 为数据源提供 `tz` 参数。这必须是与 `datetime.tzinfo` 接口兼容的对象。用户很可能会提供一个 `pytz.timezone` 实例。

注意：

用户输入的参数（如 `fromdate` 或 `sessionstart`）应与实际 `tz` 同步，无论是由数据源自动计算，用户提供还是默认（`None`，表示直接输入-输出日期时间）。

考虑到上述所有内容，让我们回顾一下柏林的交易者，交易 US/Eastern 时区的产品：

```python
import pytz
import bt

data = bt.feeds.MyFeed('ES-Mini', tz=pytz.timezone('US/Eastern'))

class Strategy(bt.Strategy):

    def next(self):
        # 这将全年有效。
        # 数据源将在 'US/Eastern' 时区的框架内返回数据，
        # 用户将 '10:00' 作为参考时间
        # 因为在 'US/Eastern' 时区，SPX 指数总是
        # 在 09:30 开盘，这将始终有效

        if self.data.datetime.time() < datetime.time(10, 0):
            # 市场运行 30 分钟之前不操作
            return  #
```

对于可以自动确定输出时区的数据源：

```python
import bt

data = bt.feeds.MyFeedAutoTZ('ES-Mini')

class Strategy(bt.Strategy):

    def next(self):
        # 这将全年有效。
        # 数据源将在 'US/Eastern' 时区的框架内返回数据，
        # 用户将 '10:00' 作为参考时间
        # 因为在 'US/Eastern' 时区，SPX 指数总是
        # 在 09:30 开盘，这将始终有效

        if self.data.datetime.time() < datetime.time(10, 0):
            # 市场运行 30 分钟之前不操作
            return  #
```

显然，上面示例中的 `MyFeed` 和 `MyFeedAuto` 只是虚拟名称。

注意：

在撰写本文时，唯一包含在发行版中的能够自动确定时区的数据源是连接到 Interactive Brokers 的数据源。
