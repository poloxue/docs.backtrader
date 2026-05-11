---
title: "平台操作"
weight: 2
---

# 启动和运行

使用数据重播功能时，同一个条会多次调用 `next` 方法，因为条的发展过程被重播。

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

`start` 和 `stop` 方法不言自明。如打印文本所示，当策略需要接收通知时，会调用 `notify_order` 方法。用例：

- 请求买入或卖出（如在 `next` 中所见）
- `buy`/`sell` 返回一个提交给经纪人的订单，是否保留该订单引用由调用者自行决定。
- 可用于确保订单处于待处理状态时不再提交新订单。
- 订单被接受/执行/取消/更改时，经纪人通过 `notify` 方法将状态变化（如执行规模）通知策略。

快速入门指南中的 `notify_order` 方法提供了一个完整的订单管理示例。

策略类还可以做更多事情：

- `buy / sell / close`

使用底层经纪人和定量器发送买入/卖出订单给经纪人，也可以手动创建订单并传递给经纪人。但平台的目的是让使用者更轻松。

`close` 获取当前市场头寸并平仓。

- `getposition`（或属性 `position`）

返回当前市场头寸。

- `setsizer/getsizer`（或属性 `sizer`）

用于设置/获取底层股份定量器（Sizer），可尝试不同定量器逻辑（固定大小、与资本成比例、指数等）。

相关文献很多，Van K. Tharp 在这方面有很好的书籍。

策略是一个线对象，支持通过标准 Python `kwargs` 参数配置：

```python
class MyStrategy(bt.Strategy):

    params = (('period', 20),)

    def __init__(self):
        self.sma = btind.SimpleMovingAverage(self.data, period=self.params.period)
    ...
```

注意，`SimpleMovingAverage` 不再以固定值 20 实例化，而是使用策略定义的 `period` 参数。

## Cerebro

数据源和策略就绪后，Cerebro 实例将它们整合并执行。实例化很简单：

```python
cerebro = bt.Cerebro()
```

没有特殊需求时，默认配置可以处理所有内容。

- 创建默认经纪人
- 无佣金操作
- 数据源预加载
- 默认执行模式为 runonce（批处理操作），速度最快

所有指标都必须支持 runonce 模式以达到全速。平台内置的指标都支持。

自定义指标无需实现 runonce 功能，Cerebro 会模拟它，但不兼容 runonce 的指标运行会更慢。不过系统大部分仍在批处理模式下运行。

由于数据源已经可用，策略也已经创建，标准的将它们结合在一起并启动运行的方法是：

```python
cerebro.adddata(data)
cerebro.addstrategy(MyStrategy, period=25)
cerebro.run()
```

请注意以下几点：

- 添加数据源”实例”
- 添加 `MyStrategy`”类”及传递给它的参数（kwargs）
- `MyStrategy` 的实例化由 Cerebro 在后台完成，`addstrategy` 中的 kwargs 会传递给实例

可以根据需要添加任意多个策略和数据源。策略如何相互通信和协调（如果需要）由用户决定，平台不强制限制。

Cerebro 还提供了额外功能：

- 决定预加载和操作模式：

```python
cerebro = bt.Cerebro(runonce=True, preload=True)
```

有一个限制：runonce 需要预加载（否则不能运行批处理操作）。但预加载数据源并不强制使用 runonce。

- `setbroker / getbroker`（和broker属性）

如果需要，可以设置自定义经纪人。也可以访问实际的经纪人实例。

- 绘图。常规情况下只需：

```python
cerebro.run()
cerebro.plot()
```

`plot` 接受一些自定义参数：

- `numfigs=1`：如果绘图过于密集，可分成多个图
- `plotter=None`：可传递自定义绘图器实例，Cerebro 将不使用默认绘图器

`**kwargs` - 标准关键字参数
将传递给绘图器。

更多信息请参阅绘图部分。

## 策略优化

如上所述，Cerebro 接收的是策略派生类（而非实例）和实例化时传递的参数，实例化在调用 `run` 时发生。

这是为了支持优化。相同的策略类可根据需要多次实例化，每次传入不同参数。如果传实例给 Cerebro...这是不可能的。

请求优化如下：

```python
cerebro.optstrategy(MyStrategy, period=xrange(10, 20))
```

`optstrategy` 方法的签名与 `addstrategy` 相同，但内部会做额外处理以确保优化正常运行。策略可能会将范围视为常规参数，而 `addstrategy` 不会对参数做任何假设。

而 `optstrategy` 能理解可迭代对象是一组值，需按顺序传递给策略类的每次实例化。

注意，传递的是一系列值而非单个值。这个例子中，策略会尝试 10 到 19 共 10 个值（20 是上限）。

如果开发了更复杂的策略，参数都可以传给 `optstrategy`。不需要优化的参数可直接传递，无需创建只有一个值的虚拟可迭代对象。例如：

```python
cerebro.optstrategy(MyStrategy, period=xrange(10, 20), factor=3.5)
```

`optstrategy` 会检测到 `factor` 并在后台为其创建虚拟可迭代对象，该对象只有一个元素（即示例中的 3.5）。

**注意**

交互式Python shell和某些类型的Windows下冻结的可执行文件在使用Python多进程模块时存在问题。

请阅读Python文档了解多进程模块。
