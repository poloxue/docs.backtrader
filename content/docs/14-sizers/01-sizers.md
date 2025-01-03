---
title: "sizers"
weight: 1
---

# Sizers

策略提供交易方法，即：`buy`、`sell` 和 `close`。让我们看看 `buy` 的签名：

```python
def buy(self, data=None, size=None, price=None, plimit=None, exectype=None, valid=None, tradeid=0, **kwargs):
```

注意，如果调用者没有指定 `size`，则 `size` 的默认值为 `None`。这就是 Sizers 发挥重要作用的地方：

- `size=None` 请求策略向其 Sizer 询问实际的头寸大小

这显然意味着策略有一个 Sizer：是的，确实如此！如果用户没有添加 Sizer，后台机制会为策略添加一个默认的 Sizer。添加到策略中的默认 Sizer 是 `SizerFix`。定义的初始行：

```python
class SizerFix(SizerBase):
    params = (('stake', 1),)
```

很容易猜到这个 Sizer 只是使用 1 个单位（无论是股票、合约等）买卖。

## 使用 Sizers

### 从 Cerebro

Sizers 可以通过 Cerebro 以两种不同的方法添加：

- `addsizer(sizercls, *args, **kwargs)`：添加一个 Sizer，将应用于添加到 cerebro 的任何策略。这就是所谓的默认 Sizer。例如：

```python
cerebro = bt.Cerebro()
cerebro.addsizer(bt.sizers.SizerFix, stake=20)  # 默认策略的 Sizer
```

- `addsizer_byidx(idx, sizercls, *args, **kwargs)`：只将 Sizer 添加到 `idx` 引用的策略中。

这个 `idx` 可以作为 `addstrategy` 的返回值获得。例如：

```python
cerebro = bt.Cerebro()
cerebro.addsizer(bt.sizers.SizerFix, stake=20)  # 默认策略的 Sizer

idx = cerebro.addstrategy(MyStrategy, myparam=myvalue)
cerebro.addsizer_byidx(idx, bt.sizers.SizerFix, stake=5)

cerebro.addstrategy(MyOtherStrategy)
```

在这个例子中：

- 系统添加了一个默认的 Sizer。这适用于所有没有分配特定 Sizer 的策略。
- 对于 `MyStrategy`，在收集其插入 `idx` 后，添加了一个特定的 Sizer（更改 `stake` 参数）。
- 系统添加了第二个策略 `MyOtherStrategy`。没有为其添加特定的 Sizer。

这意味着：

- `MyStrategy` 最终会有一个内部特定的 Sizer。
- `MyOtherStrategy` 将获得默认的 Sizer。

注意：默认并不意味着策略共享单个 Sizer 实例。每个策略都接收不同的默认 Sizer 实例。要共享单个实例，共享的 Sizer 应该是一个单例类。如何定义一个超出了 backtrader 的范围。

### 从策略

`Strategy` 类提供了一个 API：`setsizer` 和 `getsizer`（以及一个属性 `sizer`）来管理 Sizer。签名：

def setsizer(self, sizer): 它接受一个已经实例化的 Sizer
def getsizer(self): 返回当前的 Sizer 实例，sizer 是可以直接获取/设置的属性

在这种情况下，Sizer 可以例如：

- 作为参数传递给策略
- 在 `__init__` 中使用属性 `sizer` 或 `setsizer` 设置，例如：

```python
class MyStrategy(bt.Strategy):
    params = (('sizer', None),)

    def __init__(self):
        if self.p.sizer is not None:
            self.sizer = self.p.sizer
```

这将允许在与 cerebro 调用相同级别创建一个 Sizer，并将其作为参数传递给系统中的所有策略，从而有效地共享一个 Sizer。

## Sizer 开发

开发一个 Sizer 很容易：

1. 从 `backtrader.Sizer` 子类化。

这使您可以访问 `self.strategy` 和 `self.broker`，尽管在大多数情况下不需要。可以通过经纪人访问的内容：

- 数据的头寸，通过 `self.strategy.getposition(data)`
- 完整的投资组合价值，通过 `self.broker.getvalue()`

注意，这当然也可以通过 `self.strategy.broker.getvalue()` 来完成。

2. 重写方法 `_getsizing(self, comminfo, cash, data, isbuy)`。

参数：

- `comminfo`：包含数据佣金信息的 `CommissionInfo` 实例，并允许计算头寸价值、操作成本和操作佣金。
- `cash`：经纪人的当前可用现金。
- `data`：操作的目标。
- `isbuy`：对于买入操作为 `True`，对于卖出操作为 `False`。

此方法返回买入/卖出操作的期望大小。

返回的符号无关紧要，例如：如果操作是卖出操作（`isbuy` 为 `False`），方法可以返回 5 或 -5。卖出操作只会使用绝对值。

Sizer 已经去经纪人那里请求了给定数据的佣金信息、实际现金水平，并提供了操作目标数据的直接引用。

让我们定义 `FixedSize` Sizer：

```python
import backtrader as bt

class FixedSize(bt.Sizer):
    params = (('stake', 1),)

    def _getsizing(self, comminfo, cash, data, isbuy):
        return self.params.stake
```

这非常简单，因为 Sizer 不进行计算，参数只是存在。

但是该机制应该允许构建复杂的头寸系统来管理进入/退出市场时的头寸。

另一个例子：一个头寸反转器：

```python
class FixedReverser(bt.FixedSize):

    def _getsizing(self, comminfo, cash, data, isbuy):
        position = self.broker.getposition(data)
        size = self.p.stake * (1 + (position.size != 0))
        return size
```

这个 Sizer 继承了 `FixedSize`，覆盖了 `_getsizing`：

- 通过 `broker` 属性获取数据的头寸。
- 使用 `position.size` 决定是否加倍固定头寸。
- 返回计算的值。

这将解除策略决定是否反转头寸或开仓的负担，Sizer 负责控制，可以随时替换而不影响逻辑。

## 实用 Sizer 应用

不考虑复杂的头寸算法，可以使用两种不同的 Sizers 将策略从仅做多转换为多空策略。只需在 cerebro 执行中更改 Sizer，策略的行为将发生变化。一个非常简单的收盘价交叉 SMA 算法：

```python
class CloseSMA(bt.Strategy):
    params = (('period', 15),)

    def __init__(self):
        sma = bt.indicators.SMA(self.data, period=self.p.period)
        self.crossover = bt.indicators.CrossOver(self.data, sma)

    def next(self):
        if self.crossover > 0:
            self.buy()
        elif self.crossover < 0:
            self.sell()
```

注意策略并没有考虑当前头寸（通过查看 `self.position`）来决定是否实际执行买入或卖出操作。只有交叉信号被考虑。Sizer 将负责所有事务。

这个 Sizer 仅在卖出时返回非零头寸大小（如果已有头寸）：

```python
class LongOnly(bt.Sizer):
    params = (('stake', 1),)

    def _getsizing(self, comminfo, cash, data, isbuy):
      if isbuy:
          return self.p.stake

      # 卖出情况
      position = self.broker.getposition(data)
      if not position.size:
          return 0  # 如果没有持仓则不卖出

      return self.p.stake
```

将所有内容放在一起（假设已经导入 backtrader 并向系统添加了数据）：

```python
cerebro.addstrategy(CloseSMA)
cerebro.addsizer(LongOnly)
cerebro.run()
```

图表（来自源代码中的示例以测试这一点）。

![LongOnly](https://example.com/longonly.png)

多空版本只需将 Sizer 更改为上面显示的 `FixedReverser`：

```python
cerebro.addstrategy(CloseSMA)
cerebro.addsizer(FixedReverser)
cerebro.run()
```

输出图表。

![FixedReverser](https://example.com/fixedreverser.png)

注意差异：

- 交易次数翻倍。
- 现金水平从未恢复到初始值，因为策略始终在市场中。

两种方法都负面，但这只是一个示例。

## bt.Sizer 参考

```python
class backtrader.Sizer()
```

这是 Sizers 的基类。任何 Sizer 都应继承此类并覆盖 `_getsizing` 方法。

**成员属性**：

- `strategy`：由 Sizer 所在的策略设置。可以访问策略的整个 API，例如如果需要获取实际数据头寸：

```python
position = self.strategy.getposition(data)
```

- `broker`：由 Sizer 所在的策略设置。可以访问一些

复杂 Sizers 可能需要的信息，例如投资组合价值等。

```python
def _getsizing(comminfo, cash, data, isbuy)
```

此方法必须由 Sizer 的子类覆盖以提供大小功能。

**参数**：

- `comminfo`：包含数据佣金信息的 `CommissionInfo` 实例，并允许计算头寸价值、操作成本和操作佣金。
- `cash`：经纪人的当前可用现金。
- `data`：操作的目标。
- `isbuy`：对于买入操作为 `True`，对于卖出操作为 `False`。

此方法必须返回要执行的实际大小（一个整数）。如果返回 0，则不会执行任何操作。

返回值的绝对值将被使用。
