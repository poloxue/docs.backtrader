---
title: "参数说明"
weight: 2
---


# 参数说明

`Cerebro` 类的详细说明如下。

## 实例化参数

参数         | 默认值     | 说明
------------ | ---------- | --------------------------------
`preload`    | True       | 是否预加载传递给策略的数据源。
`runonce`    | True       | 以矢量化模式计算指标，提升系统性能。<br>**注：** 策略和观察者始终基于事件运行。
`live`       | False      | 如果没有数据通过 `islive` 方法报告为实时，但用户仍希望以实时模式运行，可将此参数设为 True。
`maxcpus`    | None       | 优化时使用的 CPU 核心数，默认 None（启用所有可用核心）。
`stdstats`   | True       | 如果为 True，将添加默认观察者：经纪人（现金和价值）、交易和买卖。
`oldbuysell` | False      | 当 `stdstats` 为 True 且自动添加观察者时，此开关控制买卖观察者的具体行为。
`oldtrades`  | False      | 当 `stdstats` 为 True 且自动添加观察者时，此开关控制交易观察者的具体行为。
`exactbars`  | False      | <ul style="list-style-type: none;padding-left: 0; margin-left: 0;"><li>- `False`：默认值，将 `Line` 中存储的值都保存到内存。</li><li>- `True` 或 `1`：所有 `Line` 对象的内存使用减少至计算所需的最小周期。<ul><li>如果简单移动平均线的周期为 30，则底层数据将始终有一个 30 条的运行缓冲区，以允许计算 SMA。</li><li>此设置将停用 `preload` 和 `runonce`。</li><li>使用此设置还将停用绘图。</li></ul></li><li> `-1`：策略级别的数据源和指标/操作将保留所有数据在内存中。<ul><li>如 `RSI` 通过指标 `UpDay` 计算，`UpDay` 不保留所有数据在内存中。</li><li>这允许保持绘图和预加载功能。</li><li>`runonce` 将被停用。</li></ul><li>`-2`：作为策略属性的数据源和指标将保留所有点在内存中。<ul><li>例如：`RSI` 内部使用指标 `UpDay` 进行计算，此子指标不保留所有数据在内存中。</li><li>如果在 `__init__` 中定义了 `a = self.data.close - self.data.high`，那么 `a` 不保留所有数据在内存中。</li><li>这允许保持绘图和预加载功能。</li><li>`runonce` 将被停用。</li></ul></li></ul>
`objcache`   | False      | 实验选项，用于缓存 `Line` 对象并减少其数量。示例来自 `UltimateOscillator`：<div><pre><code class="language-python" data-lang="python">bp = self.data.close - TrueLow(self.data) <br/># -> 创建另一个 TrueLow(self.data)<br/><span>tr = TrueRange(self.data)</span> </code></pre></div>如果为 True，`TrueRange` 内部的第二个 `TrueLow(self.data)` 将匹配 bp 计算中的签名并被重用。极端情况下可能使线对象超出其最小周期而导致问题，因此该功能默认禁用。
`writer`     | False      | 如果为 True，将创建一个默认的 `WriterFile`，输出到标准输出。除用户代码添加的编写器外，这个编写器也会添加到策略中。
`tradehistory` | False    | 如果为 True，将激活所有策略中每个交易的更新事件日志。也可通过策略方法 `set_tradehistory` 按策略单独设置。
`optdatas`   | True       | 如果为 True 且在优化时（且系统可预加载并使用 runonce），则数据预加载仅在主进程中完成，以节省时间和资源。测试显示，执行时间从 83 秒减少到 66 秒，提升约 20%。
`optreturn`  | True       | 如果为 True，优化结果不是完整的策略对象（包括所有数据、指标、观察者等），而是只包含以下属性的对象（与策略一致）：
`params`（或 `p`）| 无     | 策略执行时的参数。
`analyzers`  | 无         | 策略所执行的分析器。多数情况下只需分析器和参数即可评估策略性能。如需详细分析生成的值（如指标值），可关闭此选项。测试显示，执行时间提升了 13%-15%，结合 `optdatas` 总提升达 32%。
`oldsync`    | False      | 从版本 1.9.0.99 开始，多个数据（相同或不同时间框架）的同步方式已变更，以支持不同长度的数据。如需恢复以数据 0 作为系统主控的旧行为，请将此参数设为 True。
`tz`         | None       | 为策略添加全局时区。`tz` 可以是：<ul><li>`None`：策略显示的日期时间将为 UTC，此为标准行为。</li><li>`pytz` 实例：用于将 UTC 转换为所选时区。</li><li>`字符串`：将尝试实例化 `pytz` 实例。</li><li>`整数`：使用 `self.datas` 中对应数据的时区（0 表示使用 `data0` 的时区）。</li></ul>
`cheat_on_open` | False   | 调用策略的 `next_open` 方法。<br/><br/>该方法在 `next` 之前触发，且早于经纪人评估订单的时机。此时指标尚未重新计算，允许在开盘价上进行股数计算时参考前一天的指标发布订单。<br/><br/>要启用 `cheat_on_open` 订单执行，还需调用 `cerebro.broker.set_coo(True)`、实例化 `BackBroker(coo=True)`（coo 表示 `cheat-on-open`），或将 `broker_coo` 参数设为 True。除非在下文中禁用，否则 Cerebro 会自动执行此操作。
`broker_coo` | True       | 自动调用经纪人的 `set_coo` 方法并设为 True，以激活 `cheat_on_open` 执行。仅在 `cheat_on_open` 也为 True 时生效。
`quicknotify` | False     | 在传递下一价格之前立即传递经纪人通知。对回测无影响，但对实时经纪人，通知可能在条传递之前很久就已发生。设为 True 时将尽快传递通知（参见实时数据源中的 `qcheck`）。出于兼容性，默认为 False，未来可能改为 True。

## 成员方法

- `addstorecb(callback)`

添加回调函数，接收将由 `notify_store` 方法处理的消息。回调的签名必须支持以下内容：

```python
callback(msg, *args, **kwargs)
```
实际接收的 `msg`、`*args` 和 `**kwargs` 由具体实现决定（完全取决于数据源/经纪人/存储），但通常这些参数应该是可打印的，便于查看和调试。

- `notify_store(msg, *args, **kwargs)`

在 Cerebro 中接收存储通知。此方法可在 Cerebro 子类中覆盖。

- `adddatacb(callback)`

添加回调函数，接收将由 `notify_data` 方法处理的消息。回调的签名必须支持以下内容：
```python
callback(data, status, *args, **kwargs)
```

实际接收的 `*args` 和 `**kwargs` 由具体实现决定（完全取决于数据源/经纪人/存储），但通常这些参数应该是可打印的，便于查看和调试。

- `notify_data(data, status, *args, **kwargs)`

在 Cerebro 中接收数据通知。此方法可在 Cerebro 子类中覆盖。

- `adddata(data, name=None)`

添加数据源实例。如果 `name` 不为 None，会将其赋值给 `data._name`，用于显示和绘图。

- `resampledata(dataname, name=None, **kwargs)`

添加数据源用于重采样。如果 `name` 不为 None，会将其赋值给 `data._name`，用于显示和绘图。其他支持的参数（如 `timeframe`、`compression`、`todate` 等）会透明传递。

- `replaydata(dataname, name=None, **kwargs)`

添加数据源用于重放。如果 `name` 不为 None，会将其赋值给 `data._name`，用于显示和绘图。其他支持的参数（如 `timeframe`、`compression`、`todate` 等）会透明传递。

- `chaindata(*args, **kwargs)`

将多个数据源链接成一个。如果通过命名参数传递且 `name` 不为 None，会将其赋值给 `data._name`，用于显示和绘图。如果为 None，则使用第一个数据的名称。

- `rolloverdata(*args, **kwargs)`

将多个数据源链接成一个。如果通过命名参数传递且 `name` 不为 None，会将其赋值给 `data._name`，用于显示和绘图。如果为 None，则使用第一个数据的名称。其他 kwargs 会传递给 `RollOver` 类。

- `addstrategy(strategy, *args, **kwargs)`

为单次运行添加策略类，在运行时实例化。`args` 和 `kwargs` 将按原样传递给策略。返回的索引可用于引用添加的其他对象（如 `Sizer`）。

- `optstrategy(strategy, *args, **kwargs)`

为优化添加策略类，在运行时实例化。`args` 和 `kwargs` 必须是包含待测试值的可迭代对象。例如，如果策略接受参数 `period`，可按如下方式调用 `optstrategy`：

```python
cerebro.optstrategy(MyStrategy, period=(15, 25))
```

这将执行值为15和25的优化。而：

```python
cerebro.optstrategy(MyStrategy, period=range(15, 25))
```

将以 `period` 值 15 到 25（不含 25，因为 Python 的 range 是半开区间）执行 `MyStrategy`。如果某个参数只需传递一个值而不需要优化，调用如下：

```python
cerebro.optstrategy(MyStrategy, period=(15,))
```

注意 `period` 仍然作为只有一个元素的可迭代对象传递。Backtrader 仍会尝试识别以下情况：

```python
cerebro.optstrategy(MyStrategy, period=15)
```

如果可行，将创建一个内部的伪可迭代对象。

- `optcallback(cb)`

添加回调函数，在优化过程中每个策略运行时会触发调用。签名：`cb(strategy)`。

- `addindicator(indcls, *args, **kwargs)`

添加指标类到系统中。在策略中进行实例化。

- `addobserver(obscls, *args, **kwargs)`

添加观察者类到系统中。在运行时进行实例化。

- `addobservermulti(obscls, *args, **kwargs)`

添加观察者类到系统中。在运行时进行实例化。会为每个数据分别添加一次。例如，观察单个数据的买入/卖出观察者；反之，`CashValue` 观察的是系统范围的值。

- `addanalyzer(ancls, *args, **kwargs)`

添加分析器类到系统中。在运行时进行实例化。

- `addwriter(wrtcls, *args, **kwargs)`

添加编写器类到系统中。在 Cerebro 中进行实例化。

- `run(**kwargs)`

执行回测的核心方法。传递的 kwargs 会覆盖 Cerebro 实例化时的标准参数值。如果未添加数据，该方法将立即退出。返回值因是否优化而异：

**未优化：** 返回包含通过 `addstrategy` 添加的策略实例的列表。<br/>
**优化：** 返回列表的列表，每个子列表包含一次优化运行后的策略实例。

- `runstop()`

在策略内部或其他地方（包括其他线程）调用时，将尽快停止执行。

- `setbroker(broker)`

设置特定的经纪人实例，替换 Cerebro 的默认经纪人。

- `getbroker()`

返回经纪人实例。也可通过 `broker` 属性访问。

- `plot(plotter=None, numfigs=1, iplot=True, start=None, end=None, width=16, height=9, dpi=300, tight=True, use=None, **kwargs)`

绘制 Cerebro 中的策略。如果 `plotter` 为 None，会创建一个默认的 `Plot` 实例，并在实例化时传递 kwargs。<br/>
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

添加 `Sizer` 类（及参数），作为添加到 Cerebro 的所有策略的默认 Sizer。

- `addsizer_byidx(idx, sizercls, *args, **kwargs)`

按 `idx` 索引添加 `Sizer` 类。该索引与 `addstrategy` 返回的索引兼容，只有此索引引用的策略会应用该 Sizer。

- `add_signal(sigtype, sigcls, *sigargs, **sigkwargs)`

向系统添加信号，后续会添加到 `SignalStrategy` 中。

- `signal_concurrent(onoff)`

为系统添加信号并将 `concurrent` 设为 True 后，将允许并发订单。

- `signal_accumulate(onoff)`

为系统添加信号并将 `accumulate` 设为 True 后，当已有持仓时再次入场将增加头寸。

- `signal_strategy(stratcls, *args, **kwargs)`

添加支持信号的 `SignalStrategy` 子类。

- `addcalendar(cal)`

向系统添加全局交易日历。单个数据源可用独立日历覆盖全局日历。`cal` 可以是 `TradingCalendar` 实例、字符串或 `pandas_market_calendars` 实例。字符串会实例化为 `PandasMarketCalendar`（需在系统中安装 `pandas_market_calendar` 模块）。如果传递的是 `TradingCalendarBase` 的子类（而非实例），也会被实例化。

- `addtz(tz)`

也可通过参数 `tz` 完成。为策略添加全局时区，`tz` 的选项如下：

可选项    | 描述
--------- | ---------------------
None      | 策略显示的日期时间将为 UTC，此为标准行为。
pytz 实例 | 用于将 UTC 时间转换为所选时区。
字符串    | 将尝试实例化 `pytz` 实例。
整数      | 使用 `self.datas` 中对应数据的时区（0 表示使用 `data0` 的时区）。

- `add_timer(when, offset=datetime.timedelta(0), repeat=datetime.timedelta(0), weekdays=[], weekcarry=False, monthdays=[], monthcarry=True, allow=None, tzdata=None, strats=False, cheat=False, *args, **kwargs)`

安排计时器调用 `notify_timer`。参数如下：

参数        |
----------- | -------------------
`when`      | 可以是 `datetime.time` 实例（参见下方 `tzdata`）、`bt.timer.SESSION_START`（参考交易时段开始）或 `bt.timer.SESSION_END`（参考交易时段结束）。
`offset`    | 必须是 `datetime.timedelta` 实例，用于偏移 `when` 的值。与 `SESSION_START` 和 `SESSION_END` 配合使用时有实际意义，例如在交易时段开始后 15 分钟触发计时器。
`repeat`    | 必须是 `datetime.timedelta` 实例，表示首次调用后是否在同一交易时段内按指定间隔重复触发。
`weekdays`  | 排序后的可迭代对象，包含计时器可触发的天数（ISO 编码，周一为 1，周日为 7）。未指定时，计时器在所有天均有效。
`weekcarry` | 默认 False，如果为 True 但工作日未发生（如交易假期），计时器将在第二天执行（即使跨周）。
`monthdays` | 排序后的可迭代对象，包含每月应执行计时器的日期。例如每月的 15 日。未指定时，计时器在所有天均有效。
`monthcarry`| 默认 True，如果该天未发生（周末、交易假期），计时器将在下一个可用日执行。
`allow`     | 默认 None，一个回调函数，接收 `datetime.date` 实例，如果允许日期触发计时器则返回 True，否则返回 False。
`tzdata`    | 可以是 None（默认）、`pytz` 实例或数据源实例。<ul><li>`None`：按字面解释 `when`（即视为 UTC 时间处理）。</li><li>`pytz` 实例：`when` 将被视为所选时区的本地时间。</li><li>数据源实例：`when` 将被视为该数据源 `tz` 参数指定的本地时间。</li></ul>**注意**，如果 `when` 是 `SESSION_START` 或 `SESSION_END` 且 `tzdata` 为 None，将使用系统第一个数据源（`self.data0`）作为参考以确定交易时段。
`strats`    | 默认 False，同时也会调用策略的 `notify_timer`。
`cheat`     | 默认 False，如果为 True，将在经纪人有机会评估订单之前触发计时器。这允许在交易时段开始前发出基于开盘价的订单。
`*args`     | 传递给 `notify_timer` 的额外参数。
`**kwargs`  | 传递给 `notify_timer` 的额外关键字参数。返回值：创建的计时器。

- `notify_timer(timer, when, *args, **kwargs)`

接收计时器通知，其中 `timer` 是由 `add_timer` 返回的计时器，`when` 是调用时间。`args` 和 `kwargs` 是传递给 `add_timer` 的附加参数。实际的 `when` 时间可能略有延迟，因为系统无法提前调用计时器。此值是计时器的设定值，而非系统时间。

- `add_order_history(orders, notify=True)`

将订单历史直接添加到经纪人，用于性能评估。

`orders`：是一个可迭代对象（例如列表、元组、迭代器、生成器），其中的每个元素也是一个可迭代对象，包含以下子元素（支持 2 种格式）：`[datetime, size, price]` 或 `[datetime, size, price, data]`。**注意**，必须按日期时间升序排序（或生成有序元素）。

具体说明如下：

参数         | 描述
------------ | ---------------
`datetime`   | Python 的 date/datetime 实例，或格式为 `YYYY-MM-DD[THH:MM:SS[.us]]` 的字符串（方括号内为可选部分）。
`size`       | 整数（正数表示买入，负数表示卖出）。
`price`      | 浮点数或整数。
`data`       | 如果存在，可取以下值：<ul><li>`None`：使用第一个数据源作为目标。</li><li>`integer`：使用该索引（按在 Cerebro 中的添加顺序）的数据。</li><li>`string`：使用该名称的数据，例如 `cerebro.adddata(data, name=value)` 指定的名称。</li></ul>

`notify`（默认 True）：如果为 True，系统中的第一个策略将收到根据每个订单信息创建的人工订单通知。 

**注意**

这里隐含了需要将数据源添加为订单的目标。这对分析器（如回报跟踪）来说是必要的。
