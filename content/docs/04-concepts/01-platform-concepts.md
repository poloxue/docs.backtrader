---
title: "平台概念"
weight: 1
---

# 平台概念

这是一些平台概念的集合，尝试收集有助于使用平台的信息片段。

## 开始之前

所有的小代码示例都假设以下导入是可用的：

```python
import backtrader as bt
import backtrader.indicators as btind
import backtrader.feeds as btfeeds
```

**注意**

访问子模块（如指标和数据源）的另一种语法：

```python
import backtrader as bt
```

然后：

```python
thefeed = bt.feeds.OneOfTheFeeds(...)
theind = bt.indicators.SimpleMovingAverage(...)
```

#### 数据源 - 传递它们

使用平台的基础工作将通过策略完成。这些策略将接收数据源。平台终端用户无需关心接收数据源：

数据源以数组形式自动提供给策略的成员变量，并且有数组位置的快捷方式。

快速预览一个从策略派生的类声明和运行平台：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.datas[0], period=self.params.period)

    ...
cerebro = bt.Cerebro()

...

data = btfeeds.MyFeed(...)
cerebro.adddata(data)

...

cerebro.addstrategy(MyStrategy, period=30)
...
```

注意以下几点：

- 策略的 `__init__` 方法没有接收 `*args` 或 `**kwargs`（它们仍然可以使用）
- 存在一个成员变量 `self.datas`，它是一个包含至少一个项目的数组/列表/可迭代对象（否则会引发异常）

数据源添加到平台后，它们将按添加到系统的顺序出现在策略中。

**注意**

这也适用于指标，如果终端用户开发了自己的自定义指标或查看现有指标的源代码。

#### 数据源的快捷方式

`self.datas` 数组项可以通过额外的自动成员变量直接访问：

- `self.data` 目标是 `self.datas[0]`
- `self.dataX` 目标是 `self.datas[X]`

示例：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.params.period)
    ...
```

#### 省略数据源

上面的示例可以进一步简化为：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(period=self.params.period)
    ...
```

在调用 `SimpleMovingAverage` 时完全删除了 `self.data`。如果这样做，指标（在本例中为 `SimpleMovingAverage`）将接收正在创建的对象（策略）的第一个数据，即 `self.data`（也称为 `self.data0` 或 `self.datas[0]`）。

#### 几乎所有东西都是数据源

不仅数据源是数据并且可以传递。指标和操作结果也是数据。

在前面的示例中，`SimpleMovingAverage` 接收 `self.datas[0]` 作为操作输入。一个带有操作和额外指标的示例：

```python
class MyStrategy(bt.Strategy):
    params = dict(period1=20, period2=25, period3=10, period4)

    def __init__(self):
        sma1 = btind.SimpleMovingAverage(self.datas[0], period=self.p.period1)
        sma2 = btind.SimpleMovingAverage(sma1, period=self.p.period2)
        something = sma2 - sma1 + self.data.close
        sma3 = btind.SimpleMovingAverage(something, period=self.p.period3)
        greater = sma3 > sma1
        sma3 = btind.SimpleMovingAverage(greater, period=self.p.period4)
    ...
```

基本上，所有东西一旦被操作，它们都会被转换为可以作为数据源使用的对象。

#### 参数

平台中的大多数其他类都支持参数的概念。

参数及其默认值作为类属性声明（元组的元组或类字典对象）。

扫描匹配参数的关键字参数 `**kwargs`，如果找到则从 `**kwargs` 中删除并将值分配给相应的参数。

参数可以通过访问成员变量 `self.params`（简写：`self.p`）在类实例中使用。

前面的快速策略预览已经包含了参数示例，但为了冗余，重点仅放在参数上。使用元组：

```python
class MyStrategy(bt.Strategy):
    params = (('period', 20),)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.p.period)
```

使用字典：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.p.period)
```

#### 线

再次说明，平台中的大多数其他对象都是启用线的对象。从终端用户的角度来看，这意味着：

它可以保存一个或多个线系列，当将值放在图表上时，这些值将形成一条线。
线（或线系列）的一个很好的例子是由股票的收盘价形成的线。这实际上是价格演变的一个众所周知的图表表示（称为收盘线）。

平台的常规使用仅涉及访问线。前面的迷你策略示例，稍加扩展，再次派上用场：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        self.movav = btind.SimpleMovingAverage(self.data, period=self.p.period)

    def next(self):
        if self.movav.lines.sma[0] > self.data.lines.close[0]:
            print('简单移动平均线大于收盘价')
```

两个具有线的对象已被暴露：

- `self.data` 它有一个包含 `close` 属性的 `lines` 属性
- `self.movav` 是一个 `SimpleMovingAverage` 指标，它有一个包含 `sma` 属性的 `lines` 属性

**注意**

显然，线是命名的。它们也可以按声明顺序顺序访问，但这应该仅在开发指标时使用。

两个线，分别为 `close` 和 `sma`，可以查询一个点（索引 0）以比较值。

存在快捷访问线的方法：

- `xxx.lines` 可以缩短为 `xxx.l`
- `xxx.lines.name` 可以缩短为 `xxx.lines_name`

复杂对象如策略和指标提供快速访问数据的线：

- `self.data_name` 提供对 `self.data.lines.name` 的直接访问

这也适用于编号的数据变量：`self.data1_name` -> `self.data1.lines.name`

此外，线名可以直接访问：

- `self.data.close` 和 `self.movav.sma`

但这种表示法并不像前一种那样明确表示是否正在访问线。

**注意**

使用这两种后续表示法设置/分配线是不支持的。

#### 线声明

如果正在开发指标，则必须声明该指标的线。

与 `params` 一样，这次仅作为元组的类属性。字典不支持，因为它们不按插入顺序存储内容。

对于 `SimpleMovingAverage`，它将像这样完成：

```python
class SimpleMovingAverage(Indicator):
    lines = ('sma',)
    ...
```

**注意**

如果将单个字符串传递给元组，声明后需要逗号，否则元组中的每个字母将被解释为要添加到元组中的项目。这可能是 Python 语法少数错误的地方之一。

如前面的示例所示，此声明在指标中创建了一条 `sma` 线，该线可以在策略逻辑中访问（可能还可以由其他指标访问以创建更复杂的指标）。

开发时有时有必要以通用的非命名方式访问线，这就是编号访问的用武之地：

- `self.lines[0]` 指向 `self.lines.sma`

如果定义了更多的线，它们将使用索引 1、2 和更高的方式进行访问。

当然，还有额外的简写版本：

- `self.line` 指向 `self.lines[0]`
- `self.lineX` 指向 `self.lines[X]`
- `self.line_X` 指向 `self.lines[X]`

在接收数据源的对象内，可以通过编号快速访问这些数据源下的线：

- `self.dataY` 指向 `self.data.lines[Y]`
- `self.dataX_Y` 指向 `self.dataX.lines[X]`，这是 `self.datas[X].lines[Y]` 的完整简写版本

#### 访问数据源中的线

在数据源内，线也可以通过省略 `lines` 来访问。这使得处理像收盘价这样的事情更加自然。

例如：

```python
data = btfeeds.BacktraderCSVData(dataname='mydata.csv')
...

class MyStrategy(bt.Strategy):
    ...

    def next(self):
        if self.data.close[0] > 30.0:
            ...
```

这似乎比同样有效的 `if self

.data.lines.close[0] > 30.0:` 更自然。同样的道理不适用于指标，原因是：

一个指标可能有一个 `close` 属性，保存一个中间计算，后者被传递到实际也称为 `close` 的线。
在数据源的情况下，没有计算发生，因为它只是一个数据源。

#### 线长度

线有一组点，并在执行期间动态增长，因此可以通过调用标准 Python `len` 函数随时测量长度。

这适用于例如：

- 数据源
- 策略
- 指标

当数据被预加载时，数据源还应用一个附加属性：

方法 `buflen` 返回数据源可用的实际条数。

`len` 和 `buflen` 之间的区别：

- `len` 报告已处理的条数
- `buflen` 报告为数据源加载的总条数

如果两者返回相同的值，则意味着没有数据被预加载，或者条处理已消耗所有预加载条（除非系统连接到实时数据源，否则这将意味着处理结束）

#### 参数和线的继承

一种元语言支持参数和线的声明。已尽一切努力使其与标准 Python 继承规则兼容。

##### 参数继承

继承应按预期工作：

- 支持多重继承
- 基类的参数被继承
- 如果多个基类定义了相同的参数，则使用继承列表中最后一个类的默认值
- 如果在子类中重新定义相同的参数，新默认值将取代基类的值

##### 线的继承

- 支持多重继承
- 所有基类的线都被继承。由于线是命名的，如果在基类中多次使用相同的名称，则只有一个版本的线

#### 索引：0 和 -1

如前所述，线是线系列，并有一组点，这些点在连接在一起时形成一条线（如在时间轴上连接所有收盘价）。

要在常规代码中访问这些点，选择使用 0 基准方法作为当前获取/设置时刻。

策略仅获取值。指标也设置值。

在之前的快速策略示例中，`next` 方法简要显示如下：

```python
def next(self):
    if self.movav.lines.sma[0] > self.data.lines.close[0]:
        print('简单移动平均线大于收盘价')
```

逻辑通过应用索引 0 获取移动平均线的当前值和当前收盘价。

**注意**

实际上，对于索引 0 和应用逻辑/算术运算符时，可以直接进行比较，如：

```python
if self.movav.lines.sma > self.data.lines.close:
    ...
```

请参阅文档后面的运算符解释。

设置值用于开发，例如一个指标，因为指标必须设置当前输出值。

可以通过以下方式为当前获取/设置点计算 `SimpleMovingAverage`：

```python
def next(self):
  self.line[0] = math.fsum(self.data.get(0, size=self.p.period)) / self.p.period
```

访问之前设置的点已按照 Python 对访问数组/可迭代对象的 -1 定义进行建模：

- 指向数组的最后一项
- 平台将最后设置的项目（当前实时获取/设置点之前）视为 -1。

因此，比较当前收盘价与前一个收盘价是一个 0 对 -1 的事情。例如在策略中：

```python
def next(self):
    if self.data.close[0] > self.data.close[-1]:
        print('收盘价今天更高')
```

当然，逻辑上，-1 之前设置的价格将用 -2、-3 等方式访问。

#### 切片

`backtrader` 不支持线对象的切片，这是遵循 [0] 和 [-1] 索引方案的设计决定。对于常规可索引 Python 对象，你可以做：

```python
myslice = self.my_sma[0:]  # 从头到尾切片
```

但请记住，0 实际上是当前交付的值，之后没有任何东西。同样：

```python
myslice = self.my_sma[0:-1]  # 从头到尾切片
```

再一次，0 是当前值，-1 是最新（上一个）交付的值。这就是为什么在 `backtrader` 生态系统中，0 -> -1 的切片没有意义。

如果切片将来被支持，它将类似于：

```python
myslice = self.my_sma[:0]  # 从当前点向后切片到开头
```

或：

```python
myslice = self.my_sma[-1:0]  # 上一个值和当前值
```

或：

```python
myslice = self.my_sma[-3:-1]  # 从上一个值向后到倒数第三个值
```

获取切片

仍然可以获取具有最新值的数组。语法：

```python
myslice = self.my_sma.get(ago=0, size=1)  # 显示默认值
```

这将返回一个包含 1 个值（`size=1`）的数组，其中当前时刻 0 是向后查看的起点。

要从当前时间点获取 10 个值（即最后 10 个值）：

```python
myslice = self.my_sma.get(size=10)  # ago 默认为 0
```

当然，数组的排序是你期望的。最左边的值是最旧的值，最右边的值是最新的值（这是一个常规的 Python 数组，而不是线对象）。

要获取跳过当前点的最后 10 个值：

```python
myslice = self.my_sma.get(ago=-1, size=10)
```

#### 线：延迟索引

`[]` 操作符语法用于在 `next` 逻辑阶段提取单个值。线对象支持附加的表示法，通过延迟线对象在 `__init__` 阶段进行值的寻址。

假设逻辑中的兴趣是将前一个收盘价与简单移动平均线的实际值进行比较。与其在每次 `next` 迭代中手动执行，可以生成一个预先定义的线对象：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        self.movav = btind.SimpleMovingAverage(self.data, period=self.p.period)
        self.cmpval = self.data.close(-1) > self.sma

    def next(self):
        if self.cmpval[0]:
            print('前一个收盘价高于移动平均线')
```

这里使用了（延迟）表示法：

- 这提供了一个延迟了 -1 的收盘价副本。
- 比较 `self.data.close(-1) > self.sma` 生成另一个线对象，该对象返回 1（如果条件为真）或 0（如果条件为假）。

#### 线耦合

可以使用如上所示的操作符 `()` 以及延迟值来提供延迟版本的线对象。

如果语法在不提供延迟值的情况下使用，则返回 `LinesCoupler` 线对象。这是为了建立操作不同时间框架数据的指标之间的耦合。

具有不同时间框架的数据源具有不同的长度，操作它们的指标复制数据的长度。例如：

- 每年大约有 250 个条的每日数据源
- 每年有 52 个条的每周数据源

试图创建一个操作（例如）比较 2 个简单移动平均线的操作，每个操作在上述数据上将会失败。不清楚如何将每日时间框架的 250 个条与每周时间框架的 52 个条匹配。

读者可以想象在后台进行日期比较以找出日 - 周对应关系，但：

- 指标只是数学公式，没有日期时间信息
- 它们对环境一无所知，只知道如果数据提供足够的值，可以进行计算。

`()`（空调用）表示法来救场：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)

    def __init__(self):
        # data0 是每日数据
        sma0 = btind.SMA(self.data0, period=15)  # 15 天的简单移动平均线
        # data1 是每周数据
        sma1 = btind.SMA(self.data1, period=5)  # 5 周的简单移动平均线

        self.buysig = sma0 > sma1()

    def next(self):
        if self.buysig[0]:
            print('每日简单移动平均线大于每周简单移动平均线')
```

这里，较大时间框架的指标 `sma1` 被与每日时间框架耦合，使用 `sma1()`。这返回一个对象，该对象与 `sma0` 的大量条兼容，并复制 `sma1` 生成的值，有效地将 52 个每周条扩展为 250 个每日条。

#### 运算符，自然构造



为了实现“易用”目标，平台允许（在 Python 的约束范围内）使用运算符。为了进一步增强此目标，运算符的使用被分为两个阶段。

##### 阶段 1 - 运算符创建对象

即使不是明确地为此目的，之前的一个示例已经看到这一点。在像指标和策略这样的对象的初始化阶段（`__init__` 方法），运算符创建可以被操作、分配或保留以供策略逻辑的评估阶段稍后使用的对象。

再次查看 `SimpleMovingAverage` 的潜在实现，进一步分解为步骤。

在 `SimpleMovingAverage` 指标的 `__init__` 方法内部的代码可能如下：

```python
def __init__(self):
    # 求和 N 期值 - datasum 现在是一个 *Lines* 对象
    # 在使用运算符 [] 和索引 0 查询时
    # 返回当前和

    datasum = btind.SumN(self.data, period=self.params.period)

    # datasum（尽管是单行，但作为 *Lines* 对象）可以
    # 像在这种情况下那样自然地除以 int/float。
    # 实际上它可以被另一个 *Lines* 对象除。
    # 操作返回一个对象，分配给 "av"，该对象再次
    # 在使用 [0] 查询时返回当前的平均值

    av = datasum / self.params.period

    # av *Lines* 对象可以自然地分配给命名的
    # 线该指标交付。使用此
    # 指标的其他对象将可以直接访问计算结果

    self.line.sma = av
```

在策略的初始化中显示一个更完整的用例：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma = btind.SimpleMovingAverage(self.data, period=20)

        close_over_sma = self.data.close > sma
        sma_dist_to_high = self.data.high - sma

        sma_dist_small = sma_dist_to_high < 3.5

        # 不幸的是，"and" 不能在 Python 中被覆盖，
        # 因为它是语言构造而不是运算符，因此平台
        # 提供了一个函数来模拟它

        sell_sig = bt.And(close_over_sma, sma_dist_small)
```

在上述操作完成后，`sell_sig` 是一个线对象，可以在策略的逻辑中稍后使用，以指示条件是否满足。

##### 阶段 2 - 运算符忠于自然

让我们首先记住，一个策略有一个 `next` 方法，系统处理每个条时调用该方法。这是运算符实际上处于阶段 2 模式的地方。基于前面的示例：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        self.sma = sma = btind.SimpleMovingAverage(self.data, period=20)

        close_over_sma = self.data.close > sma
        self.sma_dist_to_high = self.data.high - sma

        sma_dist_small = sma_dist_to_high < 3.5

        # 不幸的是，"and" 不能在 Python 中被覆盖，
        # 因为它是语言构造而不是运算符，因此平台
        # 提供了一个函数来模拟它

        self.sell_sig = bt.And(close_over_sma, sma_dist_small)

    def next(self):

        # 尽管这看起来不像是一个“运算符”，但实际上是
        # 在测试对象是否响应 True/False

        if self.sma > 30.0:
            print('简单移动平均线大于 30.0')

        if self.sma > self.data.close:
            print('简单移动平均线高于收盘价')

        if self.sell_sig:  # if sell_sig == True: 也是有效的
            print('卖出信号为真')
        else:
            print('卖出信号为假')

        if self.sma_dist_to_high > 5.0:
            print('简单移动平均线与高点的距离大于 5.0')
```

不是一个非常有用的策略，只是一个示例。在阶段 2 中，运算符返回预期的值（如果测试为真，则为布尔值，如果与浮点数比较，则为浮点数），算术运算也返回预期值。

**注意**

请注意，比较实际上并没有使用 `[]` 操作符。这是为了进一步简化事情。

- `if self.sma > 30.0:` 比较 `self.sma[0]` 和 30.0（第一行和当前值）
- `if self.sma > self.data.close:` 比较 `self.sma[0]` 和 `self.data.close[0]`

#### 一些未覆盖的运算符/函数

Python 不允许覆盖所有内容，因此提供了一些函数来应对这些情况。

**注意**

仅在阶段 1 中使用，用于创建对象以便稍后提供值。

运算符：

- `and` -> `And`
- `or` -> `Or`

逻辑控制：

- `if` -> `If`

函数：

- `any` -> `Any`
- `all` -> `All`
- `cmp` -> `Cmp`
- `max` -> `Max`
- `min` -> `Min`
- `sum` -> `Sum`

`Sum` 实际上使用 `math.fsum` 作为底层操作，因为平台使用浮点数，应用常规求和可能会影响精度。

- `reduce` -> `Reduce`

这些实用运算符/函数对可迭代对象操作。可迭代对象中的元素可以是常规的 Python 数值类型（整数、浮点数等），也可以是具有线的对象。

生成非常愚蠢的买入信号的示例：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma1 = btind.SMA(self.data.close, period=15)
        self.buysig = bt.And(sma1 > self.data.close, sma1 > self.data.high)

    def next(self):
        if self.buysig[0]:
            pass  # 在这里做一些事情
```

显然，如果 `sma1` 高于高点，它必须高于收盘价。但重点是说明 `bt.And` 的使用。

使用 `bt.If`：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma1 = btind.SMA(self.data.close, period=15)
        high_or_low = bt.If(sma1 > self.data.close, self.data.low, self.data.high)
        sma2 = btind.SMA(high_or_low, period=15)
```

分解：

- 在 `data.close` 上生成一个周期为 15 的 `SMA`
- 然后
- `bt.If` 的值大于 `close`，返回 `low`，否则返回 `high`

请记住，在调用 `bt.If` 时没有实际值返回。它返回一个线对象，就像 `SimpleMovingAverage` 一样。

值将在系统运行时计算。

生成的 `bt.If` 线对象然后被馈送到第二个 `SMA`，该 `SMA` 有时使用低价，有时使用高价进行计算。

这些函数也接受数值。同样的示例有一个修改：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma1 = btind.SMA(self.data.close, period=15)
        high_or_30 = bt.If(sma1 > self.data.close, 30.0, self.data.high)
        sma2 = btind.SMA(high_or_30, period=15)
```

现在，第二个移动平均线使用 30.0 或高价进行计算，具体取决于 `sma` 和 `close` 的逻辑状态。

**注意**

值 30 内部被转换为一个伪可迭代对象，该对象总是返回 30。
