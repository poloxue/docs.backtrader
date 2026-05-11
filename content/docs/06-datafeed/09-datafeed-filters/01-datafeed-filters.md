---
title: "过滤器 Filters"
weight: 1
---

# 过滤器 Filters

该功能是较晚加入到 **Backtrader** 中的，为适应已有的内部结构做了一些调整。因此它在灵活性和功能完备性上可能不如预期，但在许多情况下仍然够用。

尽管实现时尝试支持即插即用的过滤器链，但受限于原有内部结构，无法始终保证。因此，有些过滤器可以链式使用，有些则不能。

## 目的

将数据源提供的值转换为不同的数据流。

该实现最初是为了简化两个明显的过滤器：**重采样** 和 **重放**，它们可通过 cerebro API 直接使用。

**重采样（cerebro.resampledata）**：改变传入数据流的时间框架和压缩比例，如 `(秒, 1)` -> `(天, 1)`。重采样过滤器会拦截并缓冲数据，直到能够提供 1 天的条形数据（在遇到第二天的 1 秒数据时触发）。

**重放（cerebro.replaydata）**：利用 1 秒分辨率的数据重建 1 天的条形数据。1 天的条形数据会被反复传递并更新，直到所有 1 秒数据都显示完毕，从而模拟实际交易日的发展。

**注意**，在日期未变化时，数据的长度（len(data)）和策略的长度保持不变。

## 工作原理

使用 `addfilter` 方法为已有数据源添加过滤器：

```python
data = MyDataFeed(dataname=myname)
data.addfilter(filter, *args, **kwargs)
cerebro.adddata(data)
```

即使与重采样或重放过滤器一起使用，也可以这样做：

```python
data = MyDataFeed(dataname=myname)
data.addfilter(filter, *args, **kwargs)
cerebro.replaydata(data)
```

## 过滤器接口

过滤器必须符合以下接口要求。首先，要是一个可调用的对象，接受如下签名：

```python
callable(data, *args, **kwargs)
```

或一个可以实例化并被调用的类，在实例化时其`__init__`方法必须支持以下签名：

```python
def __init__(self, data, *args, **kwargs)
```

`__call__`方法的签名为：

```python
def __call__(self, data, *args, **kwargs)
```

每当新的数据流值到来时，实例都会被调用。`*args` 和 `**kwargs` 与 `__init__` 中传递的参数相同。

返回值  | 描述
------- | --------------
`True`  | 数据流的内部获取循环需要重新尝试从数据源获取数据，因为长度被修改。
`False` | 即使数据已被修改（如修改了 `close` 价格），数据流的长度不变。

基于类的过滤器还可以实现一个额外方法 `last`，签名如下：

```python
def last(self, data, *args, **kwargs)
```

当数据流结束时调用，允许过滤器推送缓冲的数据。例如重采样时，一个条形数据会被缓冲直到看到下一个时间段的数据。如果数据流结束，`last` 方法提供了推送缓冲数据的机会。

**注意**

如果过滤器没有参数，且添加时也没有额外参数，签名可简化为：

```python
def __init__(self, data): ...
```

## 示例过滤器

```python
class SessionFilter(object):
    def __init__(self, data):
        pass

    def __call__(self, data):
        if data.p.sessionstart <= data.datetime.time() <= data.p.sessionend:
            # 在交易时段内
            return False  # 告诉外部数据循环，当前条形数据可以继续处理

        # 在常规交易时段外
        data.backwards()  # 从数据堆栈中移除该条形数据
        return True  # 告诉外部数据循环，必须获取新的条形数据
```

该过滤器：

- 使用 `data.p.sessionstart` 和 `data.p.sessionend` 判断 Bar 是否在交易时段。
- 在交易时段内返回 `False`，表示当前条形数据可以继续处理。
- 不在交易时段时移除条形数据并返回 `True`，表示需要获取新数据。

**注意**，`data.backwards()` 使用了 `LineBuffer` 接口，涉及 backtrader 的内部实现。

### 使用场景

有些数据源包含非交易时段的数据，这些数据可能对交易者没有意义。使用此过滤器后，只有交易时段内的条形数据才会被考虑。

**过滤器的数据伪 API**

上面的示例展示了如何通过 `data.backwards()` 从数据流中移除当前条形数据。数据源对象还提供了一些有用的伪 API 调用：

- `data.backwards(size=1, force=False)`：从数据流中移除 `size` 条数据（默认 1），通过移动逻辑指针实现。如果 `force=True`，同时移除物理存储。
- `data.forward(value=float('NaN'), size=1)`：在数据流前面添加 `size` 条数据，必要时增加物理存储，用 `value` 填充。
- `data._addtostack(bar, stash=False)`：将条形数据 `bar` 添加到堆栈。如果 `stash=False`，下一轮迭代立即处理；如果 `stash=True`，则经过完整的处理循环（包括可能被过滤器重新解析）。
- `data._save2stack(erase=False, force=False)`：将当前条形数据保存到堆栈供稍后处理。如果 `erase=True`，调用 `data.backwards()` 并传递 `force` 参数。
- `data._updatebar(bar, forward=False, ago=0)`：用 `bar` 中的值覆盖数据流中相应位置。`ago=0` 更新当前条形数据，`ago=-1` 更新前一个条形数据。

## 另一个示例：Pinkfish过滤器

这是一个可以链式使用的过滤器，特别是与重放过滤器一起使用。Pinkfish 的概念是通过每日数据执行通常需要即时数据才能完成的操作。

实现方法：

将每日条形数据分成两个部分：OHL 和 C。

这些部分与重放过滤器串联后，数据流呈现如下形式：

```
With Len X -> OHL
With Len X -> OHLC
With Len X + 1 -> OHL
With Len X + 1 -> OHLC
With Len X + 2 -> OHL
With Len X + 2 -> OHLC
...
```

逻辑：

- 接收到 OHLC 条形数据时，复制并拆解为 OHL 和 C 两个部分。
- `OHL` 条形数据的收盘价被替换为开盘、最高、最低价的平均值。
- `C` 条形数据即 “tick”，收盘价用于填充四个价格字段。
- `OHL` 部分立即加入堆栈，`C` 部分推迟处理。

该过滤器与重放过滤器配合工作，合并 OHL 和 CCCC 部分，最终输出 OHLC 条形数据。

### 使用场景

例如，当今天最大值是过去20个交易日中的最高值时，可发出“关闭”订单，并在第二次tick时执行。

```python
class DaySplitter_Close(bt.with_metaclass(bt.MetaParams, object)):
    '''
    Splits a daily bar in two parts simulating 2 ticks which will be used to
    replay the data:

      - First tick: ``OHLX``

        The ``Close`` will be replaced by the *average* of ``Open``, ``High``
        and ``Low``

        The session opening time is used for this tick

      and

      - Second tick: ``CCCC``

        The ``Close`` price will be used for the four components of the price

        The session closing time is used for this tick

    The volume will be split amongst the 2 ticks using the parameters:

      - ``closevol`` (default: ``0.5``) The value indicate which percentage, in
        absolute terms from 0.0 to 1.0, has to be assigned to the *closing*
        tick. The rest will be assigned to the ``OHLX`` tick.

    **This filter is meant to be used together with** ``cerebro.replaydata``

    '''
    params = (
        ('closevol', 0.5),  # 0 -> 1 amount of volume to keep for close
    )

    # replaying = True

    def __init__(self, data):
        self.lastdt = None

    def __call__(self, data):
        # Make a copy of the new bar and remove it from stream
        datadt = data.datetime.date()  # keep the date

        if self.lastdt == datadt:
            return False  # skip bars that come again in the filter

        self.lastdt = datadt  # keep ref to last seen bar

        # Make a copy of current data for ohlbar
        ohlbar = [data.lines[i][0] for i in range(data.size())]
        closebar = ohlbar[:]  # Make a copy for the close

        # replace close price with o-h-l average
        ohlprice = ohlbar[data.Open] + ohlbar[data.High] + ohlbar[data.Low]
        ohlbar[data.Close] = ohlprice / 3.0

        vol = ohlbar[data.Volume]  # adjust volume
        ohlbar[data.Volume] = vohl = int(vol * (1.0 - self.p.closevol))

        oi = ohlbar[data.OpenInterest]  # adjust open interst
        ohlbar[data.OpenInterest] = 0

        # Adjust times
        dt = datetime.datetime.combine(datadt, data.p.sessionstart)
        ohlbar[data.DateTime] = data.date2num(dt)

        # Ajust closebar to generate a single tick -> close price
        closebar[data.Open] = cprice = closebar[data.Close]
        closebar[data.High] = cprice
        closebar[data.Low] = cprice
        closebar[data.Volume] = vol - vohl
        ohlbar[data.OpenInterest] = oi

        # Adjust times
        dt = datetime.datetime.combine(datadt, data.p.sessionend)
        closebar[data.DateTime] = data.date2num(dt)

        # Update stream
        data.backwards(force=True)  # remove the copied bar from stream
        data._add2stack(ohlbar)  # add ohlbar to stack
        # Add 2nd part to stash to delay processing to next round
        data._add2stack(closebar, stash=True)

        return False  # initial tick can be further processed from stack
```
