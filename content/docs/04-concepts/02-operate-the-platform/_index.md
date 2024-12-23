---
title: "平台操作"
weight: 2
---

# 平台操作

## 线迭代器

为了进行操作，平台使用了线迭代器的概念。它们被松散地模拟成Python的迭代器，但实际上与它们没有任何关系。

策略和指标都是线迭代器。

线迭代器的概念试图描述以下内容：

- 一个线迭代器启动从属线迭代器，告诉它们进行迭代
- 然后，线迭代器迭代其自己声明的命名线并设置值

迭代的关键，就像常规的Python迭代器一样，是：

## next方法

- 它将在每次迭代时被调用。线迭代器拥有的并作为逻辑/计算基础的数据数组将已被平台移动到下一个索引（除非数据重播）。
- 当线迭代器的最小周期达到时调用。更多内容见下文。

但由于它们不是常规迭代器，因此存在两个附加方法：

## prenext

在达到线迭代器的最小周期之前调用。

## nextstart

在线迭代器的最小周期达到时恰好调用一次。

默认行为是将调用转发给next，但如果需要当然可以覆盖。

## 指标的额外方法

为了加速操作，指标支持一种称为runonce的批处理操作模式。它不是严格必要的（next方法就足够了），但大大减少了时间。

runonce方法规则无效化了索引为0的get/set点，并依赖于直接访问持有数据的底层数组，并传递每个状态的正确索引。

定义的方法遵循next家族的命名：

## once(self, start, end)

在最小周期达到时调用。内部数组必须在start和end之间处理，它们是基于内部数组起点的零基索引。

## preonce(self, start, end)

在最小周期达到之前调用。

## oncestart(self, start, end)

在最小周期达到时恰好调用一次。

默认行为是将调用转发给once，但如果需要当然可以覆盖。

## 最小周期

一张图片胜过千言万语，在这种情况下可能也需要一个示例。`SimpleMovingAverage`能够解释它：

```python
class SimpleMovingAverage(Indicator):
    lines = ('sma',)
    params = dict(period=20)

    def __init__(self):
        ...  # 与解释无关

    def prenext(self):
        print('prenext:: 当前周期:', len(self))

    def nextstart(self):
        print('nextstart:: 当前周期:', len(self))
        # 模拟默认行为...调用next
        self.next()

    def next(self):
        print('next:: 当前周期:', len(self))
```

实例化可能如下所示：

```python
sma = btind.SimpleMovingAverage(self.data, period=25)
```

简要解释：

假设传递给移动平均线的数据是标准数据源，其默认周期为1，即：数据源生成一个没有初始延迟的条。

然后，实例化的“period=25”移动平均线将按以下方式调用其方法：

- prenext调用24次
- nextstart调用1次（反过来调用next）
- 直到数据源耗尽，next调用n次

让我们看看杀手级指标：一个`SimpleMovingAverage`覆盖另一个`SimpleMovingAverage`。实例化可能如下所示：

```python
sma1 = btind.SimpleMovingAverage(self.data, period=25)
sma2 = btind.SimpleMovingAverage(sma1, period=20)
```

现在会发生什么：

- 与上面sma1相同
- sma2接收的一个数据源的最小周期为25，即我们的sma1，因此
- 按如下所示调用sma2方法：
  - prenext前25 + 18次，总共43次
  - 25次让sma1生成其第一个有意义的值
  - 18次积累额外的sma1值
  - 总共19个值（25次调用后1次，然后再18次）
  - nextstart然后调用1次（反过来调用next）
  - 直到数据源耗尽，next调用n次

平台在系统已经处理了44个条时调用next。

最小周期已自动调整为传入数据。

策略和指标遵循这种行为：

- 仅当达到自动计算的最小周期时才会调用next（除非初始挂钩调用nextstart）
  
**注意**

相同的规则适用于runonce批处理操作模式的preonce、oncestart和once。

**注意**

尽管不推荐，但可以操纵最小周期行为。如果需要使用`setminperiod(minperiod)`方法在策略或指标中使用。

## 启动和运行

启动和运行至少涉及3个线对象：

- 一个数据源
- 一个策略（实际上是从策略派生的类）
- 一个Cerebro（西班牙语中的大脑）

## 数据源

这些对象显然提供了通过应用计算（直接和/或使用指标）将被回测的数据。

平台提供了几个数据源：

- 多种CSV格式和通用CSV读取器
- Yahoo在线提取器
- 支持接收Pandas DataFrames和blaze对象
- 与Interactive Brokers、Visual Chart和Oanda的实时数据源

平台对数据源的内容（如时间框架和压缩）没有假设。这些值可以为信息目的和高级操作（如数据源重新采样）提供，例如将5分钟数据源转换为每日数据源。

设置Yahoo Finance数据源的示例：

```python
import backtrader as bt
import backtrader.feeds as btfeeds

...

datapath = 'path/to/your/yahoo/data.csv'

data = btfeeds.YahooFinanceCSVData(
    dataname=datapath,
    reversed=True)
```

为Yahoo显示的可选reversed参数，因为从Yahoo直接下载的CSV文件从最新日期开始，而不是从最早日期开始。

如果您的数据跨越较长时间范围，可以按如下方式限制实际加载的数据：

```python
data = btfeeds.YahooFinanceCSVData(
    dataname=datapath,
    reversed=True,
    fromdate=datetime.datetime(2014, 1, 1),
    todate=datetime.datetime(2014, 12, 31))
```

如果数据源中存在，则包括fromdate和todate。

如前所述，可以添加时间框架、压缩和名称：

```python
data = btfeeds.YahooFinanceCSVData(
    dataname=datapath,
    reversed=True,
    fromdate=datetime.datetime(2014, 1, 1),
    todate=datetime.datetime(2014, 12, 31),
    timeframe=bt.TimeFrame.Days,
    compression=1,
    name='Yahoo'
   )
```

如果数据被绘制，这些值将被使用。

## 策略（派生类）

**注意**

在继续之前，如果不希望子类化策略，请查看文档的信号部分，以获得更简化的方法。

使用平台的任何人的目标是回测数据，这在策略（派生类）内部完成。

至少需要自定义2个方法：

- `__init__`
- `next`

在初始化期间，在数据和其他计算上创建并准备好稍后应用逻辑的指标。

`next`方法稍后将被调用，以便为数据的每个条应用逻辑。

**注意**

如果传递不同时间框架（因此具有不同条数）的数据源，将为主数据调用`next`方法（第一个传递给cerebro的数据，见下文），这必须是具有较小时间框架的数据。

**注意**

如果使用数据重播功能，将为同一个条多次调用`next`方法，因为条的发展被重播。

一个基本的策略派生类：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):
        self.sma = btind.SimpleMovingAverage(self.data, period=20)

    def next(self):
        if self.sma > self.data.close:
            self.buy()
        elif self.sma < self.data.close:
            self.sell()
```

策略还有其他方法（或挂钩点），可以覆盖：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):
        self.sma = btind.SimpleMovingAverage(self.data, period=20)

    def next(self):
        if self.sma > self.data.close:
            submitted_order = self.buy()
        elif self.sma < self.data.close:
            submitted_order = self.sell()

    def start(self):
        print('回测即将开始')

    def stop(self):
        print('回测已结束')

    def notify_order(self, order):
        print('已收到新订单/更改/执行/取消通知')
```

`start`和`stop`方法应该是不言自明的。正如预期的那样，并遵循打印函数中的文本，当策略需要通知时将调用`notify_order`方法。用例：

- 请求买入或卖出（如在`next`中所见）
- `buy/sell`将返回一个提交给经纪人的订单。保持对这个提交订单的引用由调用

者决定。
- 它可以用于确保如果订单仍在待处理状态，则不提交新订单。
- 如果订单被接受/执行/取消/更改，经纪人将通过`notify`方法将状态变化（例如执行规模）通知策略。

快速入门指南在`notify_order`方法中有一个完整的功能示例来进行订单管理。

可以通过其他策略类做更多事情：

- `buy / sell / close`

使用底层经纪人和定量器发送买入/卖出订单给经纪人

同样可以通过手动创建订单并将其传递给经纪人来完成。但平台的目的是使使用它的人更容易。

`close`将获取当前市场头寸并立即关闭它。

- `getposition`（或属性“position”）

返回当前市场头寸

- `setsizer/getsizer`（或属性“sizer”）

这些允许设置/获取底层股份定量器。可以检查相同情况的不同股份定量器逻辑（固定大小、与资本成比例、指数等）

有很多文献，但Van K. Tharp在这方面有很好的书籍。

策略是一个线对象，这些对象支持参数，使用标准Python `kwargs` 参数收集：

```python
class MyStrategy(bt.Strategy):

    params = (('period', 20),)

    def __init__(self):
        self.sma = btind.SimpleMovingAverage(self.data, period=self.params.period)
    ...
```

请注意，`SimpleMovingAverage`不再以固定值20实例化，而是以策略定义的参数“period”实例化。

## Cerebro

一旦数据源可用并定义了策略，Cerebro实例就将所有内容结合在一起并执行操作。实例化一个非常简单：

```python
cerebro = bt.Cerebro()
```

如果没有特别的需求，默认值将照顾一切。

- 创建一个默认经纪人
- 操作无佣金
- 数据源将被预加载
- 默认执行模式将是runonce（批处理操作），这是最快的

所有指标都必须支持runonce模式以达到全速。平台中包含的那些支持。

自定义指标不需要实现runonce功能。Cerebro将模拟它，这意味着那些不兼容runonce的指标将运行得更慢。但系统的大部分仍将在批处理模式下运行。

由于数据源已经可用，策略也已经创建，标准的将它们结合在一起并启动运行的方法是：

```python
cerebro.adddata(data)
cerebro.addstrategy(MyStrategy, period=25)
cerebro.run()
```

请注意以下几点：

- 添加数据源“实例”
- 添加`MyStrategy`“类”及将传递给它的参数（kwargs）。
- `MyStrategy`的实例化将在后台由cerebro完成，`addstrategy`中的任何kwargs将传递给它。

用户可以根据需要添加任意多个策略和数据源。策略如何相互通信以实现协调（如果需要）不受平台的强制/限制。

当然，Cerebro提供额外的可能性：

- 决定预加载和操作模式：

```python
cerebro = bt.Cerebro(runonce=True, preload=True)
```

这里有一个限制：runonce需要预加载（如果没有，不能运行批处理操作）。当然，预加载数据源并不强制runonce。

- `setbroker / getbroker`（和broker属性）

如果需要，可以设置自定义经纪人。也可以访问实际的经纪人实例。

- 绘图。常规情况下只需：

```python
cerebro.run()
cerebro.plot()
```

`plot`接受一些参数用于自定义：

- `numfigs=1`

如果绘图过于密集，可以将其分成多个图

- `plotter=None`

可以传递一个客户绘图器实例，cerebro将不实例化默认的。

`**kwargs` - 标准关键字参数

将传递给绘图器。

请参阅绘图部分了解更多信息。

## 策略优化

如上所述，Cerebro获取一个策略派生类（不是实例）以及将在实例化时传递给它的关键字参数，这将在调用“run”时发生。

这是为了启用优化。相同的策略类将根据需要实例化多次，并带有新参数。如果将实例传递给cerebro...这将是不可能的。

请求优化如下：

```python
cerebro.optstrategy(MyStrategy, period=xrange(10, 20))
```

`optstrategy`方法的签名与`addstrategy`相同，但执行额外的内部操作以确保优化按预期运行。策略可能会将范围视为策略的常规参数，而`addstrategy`不会对传递的参数做出假设。

另一方面，`optstrategy`将理解一个可迭代对象是一组值，必须按顺序传递给策略类的每个实例化。

注意，传递的是一系列值而不是单个值。在这个简单的例子中，将为此策略尝试10个值10 -> 19（20是上限）。

如果开发了具有额外参数的更复杂策略，它们都可以传递给`optstrategy`。必须不进行优化的参数可以直接传递，无需终端用户创建只有一个值的虚拟可迭代对象。例如：

```python
cerebro.optstrategy(MyStrategy, period=xrange(10, 20), factor=3.5)
```

`optstrategy`方法看到`factor`并在后台为`factor`创建（需要的）虚拟可迭代对象，该对象只有一个元素（在示例中为3.5）。

**注意**

交互式Python shell和某些类型的Windows下冻结的可执行文件在使用Python多进程模块时存在问题。

请阅读Python文档了解多进程模块。
