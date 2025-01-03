---
title: "填充器"
weight: 4
---

# 填充器

Backtrader 的经纪模拟在订单执行时有一个默认策略：忽略交易量。这是基于两个前提：

1. 交易在流动性足够高的市场中，可以一次性完全吸收买/卖订单
2. 实际的交易量匹配需要真实的市场环境

一个简单的例子是“立即成交或取消”（Fill or Kill）订单。即使细化到每一笔交易，并且有足够的交易量来完成订单，Backtrader的经纪模拟也无法知道市场中有多少其他参与者来判断这样的订单是否会被匹配以遵循“立即成交”部分，或者订单是否应该被取消。

但是经纪模拟可以接受交易量填充器（Volume Fillers），它们决定在给定时间点应该使用多少交易量来匹配订单。

## 填充器签名

在Backtrader生态系统中，填充器可以是任何符合以下签名的可调用对象：

```python
callable(order, price, ago)
```

其中：

- `order` 是即将执行的订单，该对象提供对目标数据对象的访问，创建的大小/价格、执行的价格/大小/剩余大小和其他详细信息
- `price` 是订单执行的价格
- `ago` 是数据在订单中的索引，用于查找交易量和价格元素

在几乎所有情况下，这将是0（当前时间点），但在某些特殊情况下（例如Close订单），这可能是-1。

例如，访问bar交易量可以这样做：

```python
barvolume = order.data.volume[ago]
```

可调用对象可以是一个函数，或例如支持`__call__`方法的类的实例，例如：

```python
class MyFiller(object):
    def __call__(self, order, price, ago):
        pass
```

将填充器添加到经纪模拟

最直接的方法是使用`set_filler`：

```python
import backtrader as bt

cerebro = Cerebro()
cerebro.broker.set_filler(bt.broker.fillers.FixedSize())
```

第二种选择是完全替换经纪模拟，这可能仅适用于重写了部分功能的BrokerBack子类：

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

使用条形图中某个百分比的交易量返回给定订单的执行大小。

该百分比通过参数`perc`设置。

参数：

`size`（默认：None）最大执行大小。如果执行时间的条形图实际交易量小于该大小，则条形图的实际交易量也是一个限制。

如果该参数的值评估为False，则将使用条形图的全部交易量来匹配订单。

```python
class backtrader.fillers.FixedBarPerc()
```

使用条形图中某个百分比的交易量返回给定订单的执行大小。

该百分比通过参数`perc`设置。

参数：

`perc`（默认：100.0）（有效值：0.0 - 100.0）

用于执行订单的条形图交易量百分比

```python
class backtrader.fillers.BarPointPerc()
```

返回给定订单的执行大小。交易量将在高-低范围内均匀分布，使用`minmov`进行分区。

对于给定价格分配的交易量，将使用`perc`百分比。

参数：

`minmov`（默认：0.01）

最小价格变动。用于分区高-低范围，以按比例分配可能价格之间的交易量

`perc`（默认：100.0）（有效值：0.0 - 100.0）

用于订单执行价格匹配的分配交易量百分比
