---
title: "过滤器 Filters"
weight: 1
---

# 过滤器 Filters

该功能是较晚加入到backtrader中的，且为了适应已有的内部结构进行了一些调整。因此，它在灵活性和功能完备性上可能不如预期，但在许多情况下仍然能达到目的。

尽管实现时尝试支持即插即用的过滤器链，但由于原有内部结构的限制，始终无法保证每次都能实现。因此，有些过滤器可以链式使用，而有些则不能。

**目的**

将数据源提供的值转换为不同的数据流。

该实现最初是为了简化两个明显的过滤器的实现，这两个过滤器可以通过cerebro API直接使用，分别是：

- **重采样（cerebro.resampledata）**

  这个过滤器会改变传入数据流的时间框架和压缩比例。例如：
  
  `(秒，1)` -> `(天，1)`
  
  这意味着原始数据流是以1秒为周期的数据条。重采样过滤器会拦截数据并进行缓冲，直到能够提供1天的条形数据。这发生在看到第二天的1秒条形数据时。

- **重放（cerebro.replaydata）**

  在上面相同的时间框架下，过滤器会利用1秒的分辨率条形数据重建1天的条形数据。
  
  也就是说，1天的条形数据会被反复传递，直到显示出所有1秒的条形数据，并且数据内容会更新。

  这种方法模拟了实际交易日的发展。

**注意**

在日期没有变化的情况下，数据的长度（len(data)）以及策略的长度保持不变。

**过滤器的工作原理**

给定一个已有的数据源，你可以使用`addfilter`方法来添加过滤器：

```python
data = MyDataFeed(dataname=myname)
data.addfilter(filter, *args, **kwargs)
cerebro.adddata(data)
```

即使它与重采样或重放过滤器兼容，你也可以做如下操作：

```python
data = MyDataFeed(dataname=myname)
data.addfilter(filter, *args, **kwargs)
cerebro.replaydata(data)
```

**过滤器接口**

过滤器必须符合以下接口要求：

- 一个可调用的对象，接受如下签名：

```python
callable(data, *args, **kwargs)
```

或

- 一个可以实例化并被调用的类，在实例化时其`__init__`方法必须支持以下签名：

```python
def __init__(self, data, *args, **kwargs)
```

`__call__`方法的签名为：

```python
def __call__(self, data, *args, **kwargs)
```

每当新的数据流值到来时，实例都会被调用。`*args`和`**kwargs`与`__init__`方法传递的参数相同。

**返回值：**

- `True`：表示数据流的内部数据获取循环需要重新尝试从数据源中获取数据，因为数据流的长度被修改了。
- `False`：即使数据可能已经被编辑（例如：修改了`close`价格），数据流的长度保持不变。

如果是基于类的过滤器，还可以实现两个额外的方法：

- `last`，其签名为：

```python
def last(self, data, *args, **kwargs)
```

当数据流结束时，这个方法会被调用，允许过滤器推送它可能缓冲的数据。例如在重采样的情况下，一个条形数据会被缓冲，直到看到下一个时间段的数据。如果数据流结束，就没有新的数据可以推动缓冲的数据，`last`方法提供了推送缓冲数据的机会。

**注意**

如果过滤器没有任何参数，且在添加时没有额外的参数，签名可以简化为：

```python
def __init__(self, data) -> def __init__(self, data)
```

**示例过滤器**

以下是一个非常简单的过滤器实现：

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

- 使用`data.p.sessionstart`和`data.p.sessionend`（标准数据源参数）判断条形数据是否在交易时段内。
- 如果在交易时段内，返回`False`，表示没有做任何修改，当前条形数据可以继续处理。
- 如果不在交易时段内，条形数据会被移除，返回`True`表示需要获取新数据。

**注意**

`data.backwards()`使用了`LineBuffer`接口，深入了backtrader的内部实现。

**过滤器的使用场景：**

有些数据源包含了非交易时段的数据，这些数据可能对交易者没有意义。使用此过滤器，只有在交易时段内的条形数据才会被考虑。

**数据伪API for 过滤器**

在上面的示例中，展示了如何通过`data.backwards()`方法从数据流中移除当前条形数据。数据源对象中有一些有用的调用，作为过滤器的伪API，具体如下：

- `data.backwards(size=1, force=False)`：从数据流中移除`size`条数据（默认为1），通过将逻辑指针向后移动。如果`force=True`，则物理存储也会被移除。
  
- `data.forward(value=float('NaN'), size=1)`：将`size`条数据移到数据流的前面，如果需要会增加物理存储，并用`value`填充。

- `data._addtostack(bar, stash=False)`：将条形数据`bar`添加到堆栈中，以便以后处理。如果`stash=False`，条形数据将在下一轮迭代时立即被处理；如果`stash=True`，则会经过完整的处理循环，包括可能被过滤器重新解析。

- `data._save2stack(erase=False, force=False)`：将当前条形数据保存到堆栈中，以便稍后处理。如果`erase=True`，则会调用`data.backwards()`，并接收`force`参数。

- `data._updatebar(bar, forward=False, ago=0)`：使用`bar`中的值覆盖数据流中相应位置的数据。如果`ago=0`，则更新当前条形数据。如果`ago=-1`，则更新前一个条形数据。

**另一个示例：Pinkfish过滤器**

这是一个可以链式使用的过滤器示例，特别是与重放过滤器一起使用。Pinkfish的名字来源于该库的主页面，它的概念是通过使用每日数据来执行仅能通过即时数据完成的操作。

实现方法：

- 将每日条形数据分成两个部分：OHL和C。

这些部分与重放一起被串联，在数据流中呈现出以下形式：

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

- 当接收到一个OHLC条形数据时，会复制它，并拆解成两个部分：OHL和C。
- `OHL`条形数据的关闭价格被替换为开盘、最高和最低价格的平均值。
- `C`条形数据即为“tick”，关闭价格会用来填充四个价格字段。
- 这两个部分被分别处理，`OHL`部分会立即加入堆栈，`C`部分则被推迟处理。

该过滤器与以下功能一起工作：

- 重放过滤器，合并OHLO和CCCC部分，最终输出OHLC条形数据。

使用场景：

例如，当今天的最大值是过去20个交易日中的最高值时，可以发出“关闭”订单，并在第二次tick时执行。

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
