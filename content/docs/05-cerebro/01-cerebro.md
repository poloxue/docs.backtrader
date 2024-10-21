---
title: "Cerebro"
weight: 1
---

### Cerebro

这个类是backtrader的核心，因为它作为一个中心点用于：

- 收集所有输入（数据源）、执行者（策略）、观察者、评论者（分析器）和记录者（编写器），确保随时能够继续运行。
- 执行回测或实时数据供给/交易。
- 返回结果。
- 提供绘图功能。

#### 收集输入

首先创建一个cerebro实例：

```python
cerebro = bt.Cerebro(**kwargs)
```

一些用于控制执行的**kwargs参数支持，请参见参考（相同的参数可以稍后应用于run方法）。

#### 添加数据源

最常见的模式是`cerebro.adddata(data)`，其中`data`是已实例化的数据源。示例：

```python
data = bt.BacktraderCSVData(dataname='mypath.days', timeframe=bt.TimeFrame.Days)
cerebro.adddata(data)
```

可以重新采样和重放数据，遵循相同的模式：

```python
data = bt.BacktraderCSVData(dataname='mypath.min', timeframe=bt.TimeFrame.Minutes)
cerebro.resampledata(data, timeframe=bt.TimeFrame.Days)
```

或：

```python
data = bt.BacktraderCSVData(dataname='mypath.min', timeframe=bt.TimeFrame.Minutes)
cerebro.replaydata(data, timeframe=bt.TimeFrame.Days)
```

系统可以接受任意数量的数据源，包括混合常规数据与重新采样和/或重放的数据。当然，其中一些组合肯定没有意义，并且为了能够组合数据，需要满足一个限制：时间对齐。请参阅“数据 - 多时间框架”、“数据重新采样 - 重新采样”和“数据 - 重放”部分。

#### 添加策略

与已经是一个类实例的数据源不同，cerebro直接获取策略类和传递给它的参数。背后的理由是：在优化场景中，类将被多次实例化并传递不同的参数。

即使没有运行优化，模式仍然适用：

```python
cerebro.addstrategy(MyStrategy, myparam1=value1, myparam2=value2)
```

在优化时，必须将参数添加为可迭代对象。有关详细解释，请参阅“优化”部分。基本模式：

```python
cerebro.optstrategy(MyStrategy, myparam1=range(10, 20))
```

这将运行`MyStrategy` 10次，`myparam1`取值从10到19（记住Python中的范围是半开的，20不会被包含）。

#### 其他元素

还有一些其他元素可以添加，以增强回测体验。请参阅相应的部分。方法包括：

- `addwriter`
- `addanalyzer`
- `addobserver`（或`addobservermulti`）

#### 更改经纪人

Cerebro将使用backtrader中的默认经纪人，但可以被覆盖：

```python
broker = MyBroker()
cerebro.broker = broker  # 使用getbroker/setbroker方法的属性
```

#### 接收通知

如果数据源和/或经纪人发送通知（或创建它们的存储提供者），它们将通过`Cerebro.notify_store`方法接收。有三种（3）方法处理这些通知：

1. 通过`addnotifycallback(callback)`调用向cerebro实例添加回调。回调必须支持以下签名：

```python
callback(msg, *args, **kwargs)
```

实际接收的`msg`、`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。

2. 在添加到cerebro实例的策略子类中覆盖`notify_store`方法。签名：`notify_store(self, msg, *args, **kwargs)`。

3. 子类化Cerebro并覆盖`notify_store`（与策略中的签名相同）。这应该是最不优先的方法。

#### 执行回测

只有一个方法可以执行回测，但它支持多个选项（也可以在实例化时指定）来决定如何运行：

```python
result = cerebro.run(**kwargs)
```

请参阅下面的参考，以了解可用的参数。

#### 标准观察者

除非另有指定，cerebro会自动实例化三个标准观察者：

- 追踪现金和价值（投资组合）的经纪人观察者。
- 显示每笔交易效果的交易观察者。
- 记录操作执行时间的买入/卖出观察者。

如果希望更清晰的绘图，只需将`stdstats=False`禁用它们。

#### 返回结果

cerebro返回在回测期间创建的策略实例。这允许分析它们的操作，因为策略中的所有元素都是可访问的：

```python
result = cerebro.run(**kwargs)
```

`run`返回的结果格式将根据是否使用了优化而有所不同（策略是使用`optstrategy`添加的）：

- 所有使用`addstrategy`添加的策略。
  - `result`将是回测期间运行的实例列表。
- 使用`optstrategy`添加的一个或多个策略。
  - `result`将是一个列表的列表。每个内部列表将包含每次优化运行后的策略。

**注意**

优化的默认行为已更改为仅返回系统中存在的分析器，以使跨计算机核心的消息传递更轻量化。

如果希望将完整的策略集作为返回值，请将参数`optreturn`设置为False。

#### 提供绘图功能

如果安装了matplotlib，还可以绘制策略。通常的模式是：

```python
cerebro.plot()
```

请参阅下面的参考和绘图部分。

### 回测逻辑

流程的简要概述：

1. 传递任何存储通知。
2. 要求数据源传递下一组tick/条。
   - **版本更改**：在版本1.9.0.99中更改：新行为。
   - 数据源通过查看将由可用数据源提供的下一个日期时间进行同步。在新周期内未交易的数据源仍提供旧数据点，而有新数据可用的数据源提供此数据（以及指标的计算）。
   - 旧行为（在使用`oldsync=True`与Cerebro时保留）。
     - 插入系统的第一个数据是数据主导者，系统将等待它提供一个tick。
     - 其他数据源，或多或少，依赖于数据主导者：
       - 如果要提供的下一个tick（时间上）比数据主导者提供的tick新，将不会提供。
       - 可能由于多种原因返回而不提供新tick。
   - 该逻辑设计用于轻松同步多个数据源和不同时间框架的数据源。
3. 通知策略有关订单、交易和现金/价值的排队经纪人通知。
4. 告诉经纪人接受排队订单并使用新数据执行待处理订单。
5. 调用策略的`next`方法以让策略评估新数据（并可能发出排队到经纪人的订单）。
   - 根据阶段，可能是`prenext`或`nextstart`，在策略/指标的最小周期要求满足之前。
   - 内部策略还会触发观察者、指标、分析器和其他活动元素。
6. 告诉任何编写器将数据写入其目标。

#### 重要注意事项

**注意**

在上面的步骤1中，当数据源传递新的一组条时，这些条是关闭的。这意味着数据已经发生。

因此，策略在步骤4中发出的订单不能用步骤1中的数据执行。

这意味着订单将以x + 1的概念执行。x是订单执行的条时刻，x + 1是可能订单执行的最早时刻。

### 参考

#### 类backtrader.Cerebro()

参数：

- `preload`（默认：True）：是否预加载传递给策略的不同数据源。
- `runonce`（默认：True）：以矢量化模式运行指标以加速整个系统。策略和观察者将始终基于事件运行。
- `live`（默认：False）：如果没有数据报告为实时（通过数据的`islive`方法，但终端用户仍希望以实时模式运行，可以将此参数设置为true。
- `maxcpus`（默认：None -> 所有可用核心）：同时使用多少核心进行优化。
- `stdstats`（默认：True）：如果为True，将添加默认观察者：经纪人（现金和价值）、交易和买卖。
- `oldbuysell`（默认：False）：如果`stdstats`为True且自动添加观察者，此开关控制买卖观察者的主要行为。
- `oldtrades`（默认：False）：如果`stdstats`为True且自动添加观察者，此开关控制交易观察者的主要行为。
- `exactbars`（默认：False）：默认值下，每个存储在线中的值都保存在内存中。可能的值：
  - `True`或`1`：所有“线”对象将内存使用减少到自动计算的最小

周期。
    - 如果简单移动平均线的周期为30，则底层数据将始终有一个30条的运行缓冲区，以允许计算简单移动平均线。
    - 此设置将停用`preload`和`runonce`。
    - 使用此设置还将停用绘图。
  - `-1`：在策略级别的数据源和指标/操作将保留所有数据在内存中。
    - 例如：`RSI`内部使用指标`UpDay`进行计算。此子指标将不保留所有数据在内存中。
    - 这允许保持绘图和预加载活动。
    - `runonce`将被停用。
  - `-2`：作为策略属性的数据源和指标将保留所有点在内存中。
    - 例如：`RSI`内部使用指标`UpDay`进行计算。此子指标将不保留所有数据在内存中。
    - 如果在`__init__`中定义了`a = self.data.close - self.data.high`，那么`a`将不保留所有数据在内存中。
    - 这允许保持绘图和预加载活动。
    - `runonce`将被停用。
- `objcache`（默认：False）：实验选项，用于实现线对象的缓存并减少它们的数量。示例来自`UltimateOscillator`：
  ```python
  bp = self.data.close - TrueLow(self.data)
  tr = TrueRange(self.data)  # -> 创建另一个 TrueLow(self.data)
  ```
  如果为True，`TrueRange`内部的第二个`TrueLow(self.data)`将匹配bp计算中的签名，并将被重用。可能发生的极端情况是这会使线对象超出其最小周期并导致问题，因此被禁用。
- `writer`（默认：False）：如果设置为True，将创建一个默认的`WriterFile`，它将打印到标准输出。它将被添加到策略中（除了用户代码添加的任何其他编写器）。
- `tradehistory`（默认：False）：如果设置为True，它将激活所有策略中每个交易的更新事件日志记录。也可以通过策略方法`set_tradehistory`在每个策略基础上完成。
- `optdatas`（默认：True）：如果为True且在优化（并且系统可以预加载和使用runonce），则数据预加载将仅在主进程中完成，以节省时间和资源。测试显示，执行时间从83秒减少到66秒，约20%的速度提升。
- `optreturn`（默认：True）：如果为True，优化结果将不会是完整的策略对象（和所有数据、指标、观察者...），而是具有以下属性的对象（与策略相同）：
  - `params`（或`p`）：执行策略时的参数。
  - `analyzers`：策略已执行的分析器。
  - 在大多数情况下，仅需要分析器和参数来评估策略的性能。如果需要详细分析生成的值（例如指标），请将其关闭。测试显示，执行时间提高了13% - 15%。结合`optdatas`，总收益增加到32%。
- `oldsync`（默认：False）：从版本1.9.0.99开始，多个数据（相同或不同时间框架）的同步已更改，以允许不同长度的数据。如果希望使用数据0作为系统主控的数据的旧行为，请将此参数设置为True。
- `tz`（默认：None）：为策略添加全局时区。参数tz可以是：
  - `None`：在这种情况下，策略显示的日期时间将是UTC，这一直是标准行为。
  - `pytz`实例。将用作将UTC时间转换为所选时区。
  - `字符串`。将尝试实例化`pytz`实例。
  - `整数`。对于策略，使用`self.datas`可迭代对象中相应数据的相同时区（0将使用`data0`的时区）。
- `cheat_on_open`（默认：False）：将调用策略的`next_open`方法。这发生在`next`之前，并且在经纪人有机会评估订单之前。指标尚未重新计算。这允许发布考虑前一天指标的订单，但使用开盘价进行股份计算。对于`cheat_on_open`订单执行，还需要调用`cerebro.broker.set_coo(True)`或实例化一个`BackBroker(coo=True)`（coo表示`cheat-on-open`）或将`broker_coo`参数设置为True。除非在下文中禁用，否则Cerebro会自动执行此操作。
- `broker_coo`（默认：True）：这将自动调用经纪人的`set_coo`方法，并将其设置为True以激活`cheat_on_open`执行。只有在`cheat_on_open`也为True时才会执行。
- `quicknotify`（默认：False）：在传递下一个价格之前立即传递经纪人通知。对于回测没有影响，但对于实时经纪人，通知可能在条传递之前很久就发生。当设置为True时，将尽快传递通知（请参阅实时数据源中的`qcheck`）。为了兼容性，设置为False。可能会更改为True。
- `addstorecb(callback)`：添加一个回调以获取将由`notify_store`方法处理的消息。回调的签名必须支持以下内容：
  ```python
  callback(msg, *args, **kwargs)
  ```
  实际接收的`msg`、`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。
- `notify_store(msg, *args, **kwargs)`：在cerebro中接收存储通知。此方法可以在Cerebro子类中覆盖。实际接收的`msg`、`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。
- `adddatacb(callback)`：添加一个回调以获取将由`notify_data`方法处理的消息。回调的签名必须支持以下内容：
  ```python
  callback(data, status, *args, **kwargs)
  ```
  实际接收的`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。
- `notify_data(data, status, *args, **kwargs)`：在cerebro中接收数据通知。此方法可以在Cerebro子类中覆盖。实际接收的`*args`和`**kwargs`是实现定义的（完全依赖于数据/经纪人/存储），但通常应该期望它们是可打印的，以便接收和实验。
- `adddata(data, name=None)`：将数据源实例添加到混合中。如果`name`不为None，将放入`data._name`，用于装饰/绘图目的。
- `resampledata(dataname, name=None, **kwargs)`：添加数据源以供系统重新采样。如果`name`不为None，将放入`data._name`，用于装饰/绘图目的。任何其他如`timeframe`、`compression`、`todate`等支持的参数将透明地传递。
- `replaydata(dataname, name=None, **kwargs)`：添加数据源以供系统重放。如果`name`不为None，将放入`data._name`，用于装饰/绘图目的。任何其他如`timeframe`、`compression`、`todate`等支持的参数将透明地传递。
- `chaindata(*args, **kwargs)`：将多个数据源链接为一个。如果作为命名参数传递并且`name`不为None，将放入`data._name`，用于装饰/绘图目的。如果为None，则使用第一个数据的名称。
- `rolloverdata(*args, **kwargs)`：将多个数据源链接为一个。如果作为命名参数传递并且`name`不为None，将放入`data._name`，用于装饰/绘图目的。如果为None，则使用第一个数据的名称。任何其他kwargs将传递给`RollOver`类。
- `addstrategy(strategy, *args, **kwargs)`：为单次运行添加策略类。在运行时进行实例化。`args`和`kwargs`将在实例化期间按原样传递给策略。返回的索引可以与添加其他对象（如`Sizer`）引用兼容。
- `optstrategy(strategy, *args, **kwargs)`：为优化添加策略类。在运行时进行实例化。`args`和`kwargs`必须是包含要检查值的可迭代对象。例如：如果策略接受参数`period`，为了优化目的，调用`optstrategy`如下：
  ```python
  cerebro.optstrategy(MyStrategy, period=(15, 25))
  ```
  这将执行值

为15和25的优化。而：
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
- `optcallback(cb)`：将回调添加到回调列表中，当每个策略运行时将与优化一起调用。签名：`cb(strategy)`。
- `addindicator(indcls, *args, **kwargs)`：添加指标类到混合中。在传递策略中进行实例化。
- `addobserver(obscls, *args, **kwargs)`：添加观察者类到混合中。在运行时进行实例化。
- `addobservermulti(obscls, *args, **kwargs)`：添加观察者类到混合中。在运行时进行实例化。将为系统中的每个数据添加一次。一个用例是观察单个数据的买入/卖出观察者。一个反例是观察系统范围值的`CashValue`。
- `addanalyzer(ancls, *args, **kwargs)`：添加分析器类到混合中。在运行时进行实例化。
- `addwriter(wrtcls, *args, **kwargs)`：添加编写器类到混合中。在cerebro中进行实例化。
- `run(**kwargs)`：执行回测的核心方法。传递给它的任何kwargs将影响实例化时Cerebro的标准参数值。如果cerebro没有数据，该方法将立即退出。返回值不同：
  - 无优化：包含使用`addstrategy`添加的策略类实例的列表。
  - 优化：包含使用`addstrategy`添加的策略类实例的列表的列表。
- `runstop()`：如果从策略内部或其他地方（包括其他线程）调用，将尽快停止执行。
- `setbroker(broker)`：为此策略设置特定的经纪人实例，替换从cerebro继承的经纪人。
- `getbroker()`：返回经纪人实例。也可以通过`broker`属性访问。
- `plot(plotter=None, numfigs=1, iplot=True, start=None, end=None, width=16, height=9, dpi=300, tight=True, use=None, **kwargs)`：绘制cerebro中的策略。如果`plotter`为None，将创建一个默认的`Plot`实例，并在实例化期间将kwargs传递给它。`numfigs`将图表分成所指示的数量，以减少图表密度。`iplot`：如果为True且在笔记本中运行，图表将内联显示。`use`：将其设置为所需matplotlib后端的名称。它将优先于`iplot`。`start`：策略日期时间线数组的索引或表示绘图开始的`datetime.date`、`datetime.datetime`实例。`end`：策略日期时间线数组的索引或表示绘图结束的`datetime.date`、`datetime.datetime`实例。`width`：保存图形的宽度（以英寸为单位）。`height`：保存图形的高度（以英寸为单位）。`dpi`：保存图形的质量（以每英寸点数为单位）。`tight`：仅保存实际内容而不保存图形框架。
- `addsizer(sizercls, *args, **kwargs)`：添加一个`Sizer`类（和args），作为添加到cerebro的任何策略的默认定量器。
- `addsizer_byidx(idx, sizercls, *args, **kwargs)`：按`idx`添加一个`Sizer`类。此`idx`是与`addstrategy`返回的索引兼容的引用。只有由`idx`引用的策略将收到此大小。
- `add_signal(sigtype, sigcls, *sigargs, **sigkwargs)`：向系统添加一个信号，稍后将添加到`SignalStrategy`中。
- `signal_concurrent(onoff)`：如果将信号添加到系统并且`concurrent`值设置为True，将允许并发订单。
- `signal_accumulate(onoff)`：如果将信号添加到系统并且`accumulate`值设置为True，当已在市场中时进入市场，将允许增加头寸。
- `signal_strategy(stratcls, *args, **kwargs)`：添加可以接受信号的`SignalStrategy`子类。
- `addcalendar(cal)`：向系统添加一个全局交易日历。单个数据源可能有单独的日历覆盖全局日历。`cal`可以是`TradingCalendar`的实例、字符串或`pandas_market_calendars`的实例。字符串将实例化为`PandasMarketCalendar`（需要系统中安装模块`pandas_market_calendar`）。如果传递的是`TradingCalendarBase`的子类（而不是实例），它将被实例化。
- `addtz(tz)`：也可以使用参数`tz`完成。为策略添加全局时区。参数`tz`可以是：
  - `None`：在这种情况下，策略显示的日期时间将是UTC，这一直是标准行为。
  - `pytz`实例。将用作将UTC时间转换为所选时区。
  - `字符串`。将尝试实例化`pytz`实例。
  - `整数`。对于策略，使用`self.datas`可迭代对象中相应数据的相同时区（0将使用`data0`的时区）。
- `add_timer(when, offset=datetime.timedelta(0), repeat=datetime.timedelta(0), weekdays=[], weekcarry=False, monthdays=[], monthcarry=True, allow=None, tzdata=None, strats=False, cheat=False, *args, **kwargs)`：安排一个计时器以调用`notify_timer`。参数：
  - `when`：可以是`datetime.time`实例（见下文的`tzdata`），`bt.timer.SESSION_START`以参考会话开始，`bt.timer.SESSION_END`以参考会话结束。
  - `offset`必须是`datetime.timedelta`实例，用于偏移值`when`。在与`SESSION_START`和`SESSION_END`组合使用时具有有意义的用途，以指示例如在会话开始后15分钟调用计时器。
  - `repeat`必须是`datetime.timedelta`实例，表示在第一次调用后，是否在同一会话内按计划的重复增量安排进一步的调用。
  - `weekdays`：一个排序的可迭代对象，包含表示计时器可以实际调用的天数（iso代码，周一是1，周日是7）。如果未指定，计时器将在所有天都有效。
  - `weekcarry`（默认：False）：如果为True并且未看到工作日（例如：交易假期），计时器将在第二天执行（即使在新的一周内）。
  - `monthdays`：一个排序的可迭代对象，包含表示每月应执行计时器的天数。例如总是在每月的15日。如果未指定，计时器将在所有天都有效。
  - `monthcarry`（默认：True）：如果未看到该天（周末、交易假期），计时器将在下一个可用日执行。
  - `allow`（默认：None）：一个回调，接收一个`datetime.date`实例，如果日期被允许用于计时器则返回True，否则返回False。
  - `tzdata`可以是None（默认）、一个`pytz`实例或一个数据源实例。
    - `None`：按面值解释`when`（这意味着将其处理为UTC，即使不是）。
    - `pytz`实例：`when`将被解释为在所选时区本地时间指定。
    - 数据源实例：`when`将被解释为在数据源实例的`tz`参数指定的本地时间。
  **注意**
  如果`when`是`SESSION_START`或`SESSION_END`并且`tzdata`为None，将使用系统中的第一个数据源（即`self.data0`）作为参考，以找出会话时间。
  - `strats`（默认：False）：也调用策略的`notify_timer`。
  - `cheat`（默认：False）：如果为True，将在经纪人有机会评估订单之前调用计时器。这打开了发布基于开盘价的订单的机会，例如在会话开始前。
  - `*args`：将传递给`notify_timer`的任何额外args。
  - `**kwargs`：将传递给`notify_timer`的任何额外kwargs。
  - 返回值：创建的计时器。
- `notify_timer(timer, when, *args, **kwargs)`：接收计时器通知，其中`timer`是由`add_timer

`返回的计时器，`when`是调用时间。`args`和`kwargs`是传递给`add_timer`的任何附加参数。实际的`when`时间可能稍后，但系统可能无法提前调用计时器。此值是计时器值，而不是系统时间。
- `add_order_history(orders, notify=True)`：将订单历史直接添加到经纪人以进行性能评估。
  - `orders`：是一个可迭代对象（例如：列表、元组、迭代器、生成器），其中每个元素也是一个具有以下子元素（2种格式可能）的可迭代对象（具有长度）：
    - `[datetime, size, price]`或`[datetime, size, price, data]`。
  **注意**
  必须按日期时间升序排序（或生成排序的元素）。其中：
    - `datetime`是Python的日期/日期时间实例或格式为`YYYY-MM-DD[THH:MM:SS[.us]]`的字符串，其中括号内的元素是可选的。
    - `size`是一个整数（正数表示买入，负数表示卖出）。
    - `price`是一个浮点数/整数。
    - `data`如果存在，可以取以下任何值：
      - `None` - 将使用第一个数据源作为目标。
      - `integer` - 将使用该索引（在Cerebro中插入顺序）的数据。
      - `string` - 将使用具有该名称的数据，例如使用`cerebro.adddata(data, name=value)`分配的名称。
  - `notify`（默认：True）：如果为True，系统中插入的第一个策略将收到根据每个订单的信息创建的人工订单通知。 
**注意**
  描述中隐含的是添加作为订单目标的数据源。例如，这是分析器（如跟踪回报）的必要条件。
