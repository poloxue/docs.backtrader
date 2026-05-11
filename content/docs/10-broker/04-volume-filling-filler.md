---
title: "填充器"
weight: 4
---

# 填充器

Backtrader 的 Broker 模拟在订单执行时有一个默认策略：忽略成交量。这是基于两个假设：

1. 交易在流动性足够高的市场中进行，可一次性完全吸收买卖订单。
2. 实际的成交量匹配需要真实市场环境。

一个简单的例子是”立即成交或取消”（Fill or Kill）订单。即使逐笔细化且有足够的成交量，Backtrader 的 Broker 模拟也无法知道市场中有多少其他参与者，无法判断这样的订单是否能立即成交或应取消。

但 Broker 模拟可以接受成交量填充器（Volume Fillers），由它们决定在给定时间点使用多少成交量来匹配订单。

## 填充器签名

在 Backtrader 生态系统中，填充器可以是任何符合以下签名的可调用对象：

```python
callable(order, price, ago)
```

其中：

- `order`：即将执行的订单，提供对目标数据对象的访问，包括创建的大小/价格、执行的价格/大小/剩余大小等。
- `price`：订单执行的价格。
- `ago`：数据在订单中的索引，用于查找成交量和价格数据。

在大多数情况下，`ago` 为 0（当前时间点），但在某些特殊情况下（如 Close 订单），可能为 -1。

例如，访问 bar 的成交量：

```python
barvolume = order.data.volume[ago]
```

可调用对象可以是一个函数，或例如支持`__call__`方法的类的实例，例如：

```python
class MyFiller(object):
    def __call__(self, order, price, ago):
        pass
```

将填充器添加到 Broker 模拟

最直接的方式是使用 `set_filler`：

```python
import backtrader as bt

cerebro = Cerebro()
cerebro.broker.set_filler(bt.broker.fillers.FixedSize())
```

第二种方式是完全替换 Broker 模拟，这通常用于重写了部分功能的 BrokerBack 子类：

```python
import backtrader as bt

cerebro = Cerebro()
filler = bt.broker.fillers.FixedSize()
newbroker = bt.broker.BrokerBack(filler=filler)
cerebro.broker = newbroker
```

## 示例

Backtrader的源代码中包含一个名为`volumefilling`的示例，它允许测试一些集成的填充器（最初是全部）。

## 参考

```python
class backtrader.fillers.FixedSize()
```

使用 bar 中一定百分比的成交量返回订单的执行大小，百分比由参数 `perc` 设定。

参数：

`size`（默认：None）：最大执行大小。如果执行时 bar 的实际成交量小于该值，则以实际成交量为限。

如果该参数的值为 False，则使用 bar 的全部成交量来匹配订单。

```python
class backtrader.fillers.FixedBarPerc()
```

使用 bar 中一定百分比的成交量返回订单的执行大小，百分比通过 `perc` 参数设置。

参数：

`perc`（默认：100.0）（有效值：0.0 - 100.0）

用于执行订单的 bar 成交量百分比

```python
class backtrader.fillers.BarPointPerc()
```

返回给定订单的执行大小。成交量将在高-低范围内均匀分布，使用 `minmov` 进行分区。

对于给定价格分配的成交量，使用 `perc` 百分比。

参数：

`minmov`（默认：0.01）：最小价格变动。用于分区高-低范围，按比例分配各价位之间的成交量。

`perc`（默认：100.0）（有效值：0.0 - 100.0）：用于订单执行价格匹配的分配成交量百分比。
