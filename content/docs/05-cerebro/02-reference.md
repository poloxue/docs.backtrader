---
title: "参数说明"
weight: 2
---


# 参数说明

如下是 `Cerebro` 类的详细说明。

## 实例化参数

参数         | 默认值     | 说明
------------ | ---------- | --------------------------------
`preload`    | True       | 是否预加载传递给策略的不同数据源。
`runonce`    | True       | 以矢量化模式计算指标，从而提高整个系统的性能。<br>**注：** 策略和观察者将始终基于事件运行。
`live`       | False      | 如果没有数据报告为实时（通过数据的`islive`方法，但用户仍希望以实时模式运行，可将此参数设为true。
`maxcpus`    | None       | 同时使用多少核心进行优化，默认 None，即启用所有可用的 CPU 核。
`stdstats`   | True）     | 如果为True，将添加默认观察者：经纪人（现金和价值）、交易和买卖。
`oldbuysell` | False      | 如 `stdstats` 为 True 且自动添加观察者，此开关控制买卖观察者的主要行为。
`oldtrades`  | False      | 如果`stdstats`为True 且自动添加观察者，此开关控制交易观察者的主要行为。
`exactbars`  | False      | <ul style="list-style-type: none;padding-left: 0; margin-left: 0;"><li>- `False`：默认值，将存储在 `Line` 中的值都保存到内存。</li><li>-`True`或`1`：所有`Line`对象的内存使用减少至计算最小周期。<ul><li>如果简单移动平均线的周期为30，则底层数据将始终有一个30条的运行缓冲区，以允许计算简单移动平均线。</li><li>此设置将停用`preload`和`runonce`。</li><li>使用此设置还将停用绘图。</li></ul></li><li> `-1`：策略级别的数据源和指标/操作将保留所有数据在内存中。<ul><li>如 `RSI` 通过指标`UpDay`计算，不会将它的所有数据保留内存。</li><li> 这允许保持绘图和预加载活动。</li><li>`runonce`将被停用。</li></ul><li>`-2`：作为策略属性的数据源和指标将保留所有点在内存中。<ul><li>例如：`RSI`内部使用指标`UpDay`进行计算。此子指标将不保留所有数据在内存中。</li><li>如果在`__init__`中定义了`a = self.data.close - self.data.high`，那么`a`将不保留所有数据在内存中。</li><li>- 这允许保持绘图和预加载活动。</li><li>`runonce`将被停用。</li></ul></li></ul>
`objcache`   | False      | 实验选项，用于实现 `Line` 对象的缓存并减少它们的数量。示例来自`UltimateOscillator`：<div><pre><code class="language-python" data-lang="python">bp = self.data.close - TrueLow(self.data) <br/># -> 创建另一个 TrueLow(self.data)<br/><span>tr = TrueRange(self.data)</span> </code></pre></div>如果为True，`TrueRange`内部的第二个`TrueLow(self.data)`将匹配bp计算中的签名，并将被重用。可能发生的极端情况是这会使线对象超出其最小周期并导致问题，因此被禁用。
`writer`     | False      | 如果设置为True，将创建一个默认的`WriterFile`，它将打印到标准输出。它将被添加到策略中（除了用户代码添加的任何其他编写器）。
`tradehistory` | False    | 如果设置为True，它将激活所有策略中每个交易的更新事件日志记录。也可以通过策略方法`set_tradehistory`在每个策略基础上完成。
`optdatas`   | True       | 如果为 True 且在优化（并且系统可以预加载和使用runonce），则数据预加载将仅在主进程中完成，以节省时间和资源。测试显示，执行时间从83秒减少到66秒，约20%的速度提升。
`optreturn`  | True       | 如果为True，优化结果将不会是完整的策略对象（和所有数据、指标、观察者...），而是具有以下属性的对象（与策略相同）：
`params`（或`p`）| 无     | 执行策略时的参数。
`analyzers`  | 无         | 策略已执行的分析器。大多数情况下，仅需要分析器和参数来评估策略的性能。如果需要详细分析生成的值（例如指标），请将其关闭。测试显示，执行时间提高了13% - 15%。结合`optdatas`，总收益增加到32%。
`oldsync`    | False      | 从版本1.9.0.99开始，多个数据（相同或不同时间框架）的同步已更改，以允许不同长度的数据。如果希望使用数据0作为系统主控的数据的旧行为，请将此参数设置为True。
`tz`         | None       | 为策略添加全局时区。参数tz可以是：<ul><li>`None`：在这种情况下，策略显示的日期时间将是UTC，这一直是标准行为。</li><li>`pytz`实例。将用作将UTC时间转换为所选时区。</li><li>`字符串`。将尝试实例化`pytz`实例。</li><li>`整数`。对于策略，使用`self.datas`可迭代对象中相应数据的相同时区（0将使用`data0`的时区）。</li></ul>
`cheat_on_open` | False   | 将调用策略的`next_open`方法。<br/><br/>这发生在`next`之前，并且在经纪人有机会评估订单之前。指标尚未重新计算。这允许发布考虑前一天指标的订单，但使用开盘价进行股份计算。<br/><br/>对于`cheat_on_open`订单执行，还要调用 `cerebro.broker.set_coo(True)`或实例化一个`BackBroker(coo=True)`（coo表示`cheat-on-open`）或将`broker_coo`参数设置为True。除非在下文中禁用，否则Cerebro会自动执行此操作。
`broker_coo` | True       | 这将自动调用经纪人的`set_coo`方法，并将其设置为True以激活`cheat_on_open`执行。只有在`cheat_on_open`也为True时才会执行。
`quicknotify` | False     | 在传递下一个价格之前立即传递经纪人通知。对于回测没有影响，但对于实时经纪人，通知可能在条传递之前很久就发生。当设置为True时，将尽快传递通知（请参阅实时数据源中的`qcheck`）。为了兼容性，设置为False。可能会更改为True。

## 成员方法

- `addstorecb(callback)`

添加一个回调以获取将由`notify_store`方法处理的消息。回调的签名必须支持以下内容：

```python
callback(msg, *args, **kwargs)
```
实际接收的`msg`、`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。

- `notify_store(msg, *args, **kwargs)`

在cerebro中接收存储通知。此方法可以在Cerebro子类中覆盖。实际接收的`msg`、`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。

- `adddatacb(callback)`

添加一个回调以获取将由`notify_data`方法处理的消息。回调的签名必须支持以下内容：
```python
callback(data, status, *args, **kwargs)
```

实际接收的`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。

- `notify_data(data, status, *args, **kwargs)`

在cerebro中接收数据通知。此方法可以在Cerebro子类中覆盖。实际接收的`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。

- `adddata(data, name=None)`

将数据源实例添加到混合中。如果`name`不为None，将放入`data._name`，用于装饰/绘图目的。

- `resampledata(dataname, name=None, **kwargs)`

添加数据源以供系统重新采样。如果`name`不为None，将放入`data._name`，用于装饰/绘图目的。任何其他如`timeframe`、`compression`、`todate`等支持的参数将透明地传递。

- `replaydata(dataname, name=None, **kwargs)`

添加数据源以供系统重放。如果`name`不为None，将放入`data._name`，用于装饰/绘图目的。任何其他如`timeframe`、`compression`、`todate`等支持的参数将透明地传递。

- `chaindata(*args, **kwargs)`

将多个数据源链接为一个。如果作为命名参数传递并且`name`不为None，将放入`data._name`，用于装饰/绘图目的。如果为None，则使用第一个数据的名称。

- `rolloverdata(*args, **kwargs)`

将多个数据源链接为一个。如果作为命名参数传递并且`name`不为None，将放入`data._name`，用于装饰/绘图目的。如果为None，则使用第一个数据的名称。任何其他kwargs将传递给`RollOver`类。

- `addstrategy(strategy, *args, **kwargs)`

为单次运行添加策略类。在运行时进行实例化。`args`和`kwargs`将在实例化期间按原样传递给策略。返回的索引可以与添加其他对象（如`Sizer`）引用兼容。

- `optstrategy(strategy, *args, **kwargs)`

为优化添加策略类。在运行时进行实例化。`args`和`kwargs`必须是包含要检查值的可迭代对象。例如：如果策略接受参数`period`，为了优化目的，调用`optstrategy`如下：

```python
cerebro.optstrategy(MyStrategy, period=(15, 25))
```

这将执行值为15和25的优化。而：

```python
cerebro.optstrategy(MyStrategy, period=range(15, 25))
```

将以`period`值15 -> 25（25不包含，因为Python中的范围是半开的）执行`MyStrategy`。如果传递了一个参数但不应优化，调用如下：

```python
cerebro.optstrategy(MyStrategy, period=(15,))
```

注意`period`仍然作为一个可迭代对象传递……只有一个元素。backtrader无论如何都会尝试识别如下情况：

```python
cerebro.optstrategy(MyStrategy, period=15)
```

如果可能，将创建一个内部伪可迭代对象。

- `optcallback(cb)`

将回调添加到回调列表中，当每个策略运行时将与优化一起调用。签名：`cb(strategy)`。

- `addindicator(indcls, *args, **kwargs)`

添加指标类到混合中。在传递策略中进行实例化。

- `addobserver(obscls, *args, **kwargs)`

添加观察者类到混合中。在运行时进行实例化。

- `addobservermulti(obscls, *args, **kwargs)`

添加观察者类到混合中。在运行时进行实例化。将为系统中的每个数据添加一次。一个用例是观察单个数据的买入/卖出观察者。一个反例是观察系统范围值的`CashValue`。

- `addanalyzer(ancls, *args, **kwargs)`

添加分析器类到混合中。在运行时进行实例化。

- `addwriter(wrtcls, *args, **kwargs)`

添加编写器类到混合中。在cerebro中进行实例化。

- `run(**kwargs)`

执行回测的核心方法。传递给它的任何kwargs将影响实例化时Cerebro的标准参数值。如果cerebro没有数据，该方法将立即退出。返回值不同：

**无优化：** 包含使用`addstrategy`添加的策略类实例的列表。<br/>
**优化：** 包含使用`addstrategy`添加的策略类实例的列表的列表。

- `runstop()`

如果从策略内部或其他地方（包括其他线程）调用，将尽快停止执行。

- `setbroker(broker)`

为此策略设置特定的经纪人实例，替换从cerebro继承的经纪人。

- `getbroker()`

返回经纪人实例。也可以通过`broker`属性访问。

- `plot(plotter=None, numfigs=1, iplot=True, start=None, end=None, width=16, height=9, dpi=300, tight=True, use=None, **kwargs)`

绘制cerebro中的策略。如果`plotter`为None，将创建一个默认的`Plot`实例，并在实例化期间将kwargs传递给它。<br/>
```bash
`numfigs`将图表分成所指示的数量，以减少图表密度。<br/>
`iplot`：如果为True且在笔记本中运行，图表将内联显示。<br/>
`use`：将其设置为所需matplotlib后端的名称。它将优先于`iplot`。<br/>
`start`：策略日期时间线数组的索引或表示绘图开始的`datetime.date`、`datetime.datetime`实例。<br/>
`end`：策略日期时间线数组的索引或表示绘图结束的`datetime.date`、`datetime.datetime`实例。<br/>
`width`：保存图形的宽度（以英寸为单位）。<br/>
`height`：保存图形的高度（以英寸为单位）。<br/>
`dpi`：保存图形的质量（以每英寸点数为单位）。<br/>
`tight`：仅保存实际内容而不保存图形框架。
```

- `addsizer(sizercls, *args, **kwargs)`

添加一个`Sizer`类（和args），作为添加到cerebro的任何策略的默认定量器。

- `addsizer_byidx(idx, sizercls, *args, **kwargs)`

按`idx`添加一个`Sizer`类。此`idx`是与`addstrategy`返回的索引兼容的引用。只有由`idx`引用的策略将收到此大小。

- `add_signal(sigtype, sigcls, *sigargs, **sigkwargs)`

向系统添加一个信号，稍后将添加到`SignalStrategy`中。

- `signal_concurrent(onoff)`

如果将信号添加到系统并且`concurrent`值设置为True，将允许并发订单。

- `signal_accumulate(onoff)`

如果将信号添加到系统并且`accumulate`值设置为True，当已在市场中时进入市场，将允许增加头寸。

- `signal_strategy(stratcls, *args, **kwargs)`

添加可以接受信号的`SignalStrategy`子类。

- `addcalendar(cal)`

向系统添加一个全局交易日历。单个数据源可能有单独的日历覆盖全局日历。`cal`可以是`TradingCalendar`的实例、字符串或`pandas_market_calendars`的实例。字符串将实例化为`PandasMarketCalendar`（需要系统中安装模块`pandas_market_calendar`）。如果传递的是`TradingCalendarBase`的子类（而不是实例），它将被实例化。

- `addtz(tz)`

也可以使用参数`tz`完成。为策略添加全局时区。参数`tz`可以是：

可选项    | 描述
--------- | ---------------------
None      | 在这种情况下，策略显示的日期时间将是UTC，这一直是标准行为。
pytz 实例 | 将用作将UTC时间转换为所选时区。
字符串    | 将尝试实例化`pytz`实例。
整数      | 对于策略，使用`self.datas`中相应数据的时区（0就用`data0`的时区）。

- `add_timer(when, offset=datetime.timedelta(0), repeat=datetime.timedelta(0), weekdays=[], weekcarry=False, monthdays=[], monthcarry=True, allow=None, tzdata=None, strats=False, cheat=False, *args, **kwargs)`

安排一个计时器以调用`notify_timer`。参数：

参数        |
----------- | -------------------
`when`      | 可以是`datetime.time`实例（见下文的`tzdata`），`bt.timer.SESSION_START`以参考会话开始，`bt.timer.SESSION_END`以参考会话结束。
`offset`    | 必须是`datetime.timedelta`实例，用于偏移值`when`。在与`SESSION_START`和`SESSION_END`组合使用时具有有意义的用途，以指示例如在会话开始后15分钟调用计时器。
`repeat`    | 必须是`datetime.timedelta`实例，表示在第一次调用后，是否在同一会话内按计划的重复增量安排进一步的调用。
`weekdays`  | 一个排序的可迭代对象，包含表示计时器可以实际调用的天数（iso代码，周一是1，周日是7）。如果未指定，计时器将在所有天都有效。
`weekcarry` | 默认：False，如果为True并且未看到工作日（例如：交易假期），计时器将在第二天执行（即使在新的一周内）。
`monthdays` | 一个排序的可迭代对象，包含表示每月应执行计时器的天数。例如总是在每月的15日。如果未指定，计时器将在所有天都有效。
`monthcarry`| 默认：True，如果未看到该天（周末、交易假期），计时器将在下一个可用日执行。
`allow`     | 默认：None，一个回调，接收一个`datetime.date`实例，如果日期被允许用于计时器则返回True，否则返回False。
`tzdata`    | 可以是None（默认）、一个`pytz`实例或一个数据源实例。<ul><li> `None`：按面值解释`when`（这意味着将其处理为UTC，即使不是）。</li><li>- `pytz`实例：`when`将被解释为在所选时区本地时间指定。</li><li>数据源实例：`when`将被解释为在数据源实例的`tz`参数指定的本地时间。</li></ul>**注意**，如果`when`是`SESSION_START`或`SESSION_END`并且`tzdata`为None，将使用系统中的第一个数据源（即`self.data0`）作为参考，以找出会话时间。
`strats`    | 默认：False，也调用策略的`notify_timer`。
`cheat`     | 默认：False）如果为True，将在经纪人有机会评估订单之前调用计时器。这打开了发布基于开盘价的订单的机会，例如在会话开始前。
`*args`     | 将传递给`notify_timer`的任何额外args。
`**kwargs`  | 将传递给`notify_timer`的任何额外kwargs。返回值：创建的计时器。

- `notify_timer(timer, when, *args, **kwargs)`

接收计时器通知，其中`timer`是由 `add_timer` 返回的计时器，`when`是调用时间。`args`和`kwargs`是传递给`add_timer`的任何附加参数。实际的`when`时间可能稍后，但系统可能无法提前调用计时器。此值是计时器值，而不是系统时间。

- `add_order_history(orders, notify=True)`

将订单历史直接添加到经纪人以进行性能评估。

`orders`：是一个可迭代对象（例如：列表、元组、迭代器、生成器），其中每个元素也是一个具有以下子元素（2种格式可能）的可迭代对象（具有长度）：`[datetime, size, price]`或`[datetime, size, price, data]`，**注意**，必须按日期时间升序排序（或生成排序的元素）。

具体说明如下：

参数         | 描述
------------ | ---------------
`datetime`   | 是Python的日期/日期时间实例或格式为`YYYY-MM-DD[THH:MM:SS[.us]]`的字符串，其中括号内的元素是可选的。
`size`       | 是一个整数（正数表示买入，负数表示卖出）。
`price`      | 是一个浮点数/整数。
`data`       | 如果存在，可以取以下任何值：<ul><li>`None` - 将使用第一个数据源作为目标。</li><li> `integer` - 将使用该索引（在Cerebro中插入顺序）的数据。</li><li>`string` - 将使用具有该名称的数据，例如使用`cerebro.adddata(data, name=value)`分配的名称。</li></ul>

`notify`（默认：True）：如果为True，系统中插入的第一个策略将收到根据每个订单的信息创建的人工订单通知。 

**注意**

描述中隐含的是添加作为订单目标的数据源。例如，这是分析器（如跟踪回报）的必要条件。
