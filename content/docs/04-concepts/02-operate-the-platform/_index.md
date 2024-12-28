---
title: "平台操作"
weight: 2
---

# 启动和运行

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
