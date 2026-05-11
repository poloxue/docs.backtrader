---
title: "sizers"
weight: 1
---

# Sizers

策略提供交易方法：`buy`、`sell` 和 `close`。先看 `buy` 的签名：

```python
def buy(self, data=None, size=None, price=None, plimit=None, exectype=None, valid=None, tradeid=0, **kwargs):
```

注意，如果调用者没有指定 `size`，则 `size` 的默认值为 `None`。这就是 Sizer 发挥作用的地方：

- `size=None` 表示策略向其 Sizer 询问实际的头寸大小

这意味着策略有一个 Sizer：是的，确实如此！如果用户没有添加 Sizer，后台会为策略添加一个默认的 Sizer。默认的 Sizer 是 `SizerFix`。其定义如下：

```python
class SizerFix(SizerBase):
    params = (('stake', 1),)
```

可以猜到，这个 Sizer 只是使用 1 个单位（无论是股票、合约等）进行买卖。

## 使用 Sizers

### 通过 Cerebro

Sizers 可以通过 `Cerebro` 的两种方法添加：

- `addsizer(sizercls, *args, **kwargs)`：添加一个 Sizer，适用于所有添加到 `cerebro` 的策略。这就是所谓的默认 Sizer。例如：

```python
cerebro = bt.Cerebro()
cerebro.addsizer(bt.sizers.SizerFix, stake=20)  # 默认策略的 Sizer
```

- `addsizer_byidx(idx, sizercls, *args, **kwargs)`：将 Sizer 添加到 `idx` 引用的策略中。

这个 `idx` 可以通过 `addstrategy` 的返回值获取。例如：

```python
cerebro = bt.Cerebro()
cerebro.addsizer(bt.sizers.SizerFix, stake=20)  # 默认策略的 Sizer

idx = cerebro.addstrategy(MyStrategy, myparam=myvalue)
cerebro.addsizer_byidx(idx, bt.sizers.SizerFix, stake=5)

cerebro.addstrategy(MyOtherStrategy)
```

在这个例子中：

- 系统添加了一个默认 Sizer，适用于所有没有分配特定 Sizer 的策略。
- `MyStrategy` 在获取其 `idx` 后，添加了一个特定的 Sizer（更改了 `stake` 参数）。
- 第二个策略 `MyOtherStrategy` 没有特定的 Sizer。

这意味着：

- `MyStrategy` 最终拥有一个内部特定的 Sizer。
- `MyOtherStrategy` 使用默认 Sizer。

注意：默认并不意味着策略共享同一个 Sizer 实例。每个策略都获得不同的默认 Sizer 实例。要共享单个实例，Sizer 需要是一个单例类。如何定义单例超出了 backtrader 的范围。

### 从策略

`Strategy` 类提供了 `setsizer` 和 `getsizer` 方法（以及 `sizer` 属性）来管理 Sizer。签名：

def setsizer(self, sizer): 接受一个已实例化的 Sizer
def getsizer(self): 返回当前的 Sizer 实例，sizer 是可以直接获取/设置的属性

在这种情况下，Sizer 可以：

- 作为参数传递给策略
- 在 `__init__` 中使用 `sizer` 属性或 `setsizer` 设置，例如：

```python
class MyStrategy(bt.Strategy):
    params = (('sizer', None),)

    def __init__(self):
        if self.p.sizer is not None:
            self.sizer = self.p.sizer
```

这样可以在与 `cerebro` 调用相同的级别创建 Sizer，并将其作为参数传递给所有策略，从而实现 Sizer 共享。

## Sizer 开发

开发一个 Sizer 很简单：

1. 从 `backtrader.Sizer` 子类化。

这使你可以访问 `self.strategy` 和 `self.broker`，尽管大多数情况下不需要。可以通过经纪商访问的内容：

- 数据的头寸，通过 `self.strategy.getposition(data)`
- 完整的投资组合价值，通过 `self.broker.getvalue()`

也可以通过 `self.strategy.broker.getvalue()` 实现同样的效果。

2. 重写 `_getsizing(self, comminfo, cash, data, isbuy)` 方法。

参数：

- `comminfo`：包含数据佣金信息的 `CommissionInfo` 实例，可用于计算头寸价值、操作成本和操作佣金。
- `cash`：经纪商的当前可用现金。
- `data`：操作的目标。
- `isbuy`：买入操作为 `True`，卖出操作为 `False`。

此方法返回买入/卖出操作的期望数量。

返回值的符号无关紧要。例如：如果是卖出操作（`isbuy` 为 `False`），方法可以返回 5 或 -5。卖出操作只使用绝对值。

Sizer 已经向经纪商请求了给定数据的佣金信息和实际现金水平，并提供了操作目标数据的直接引用。

让我们定义 `FixedSize` Sizer：

```python
import backtrader as bt

class FixedSize(bt.Sizer):
    params = (('stake', 1),)

    def _getsizing(self, comminfo, cash, data, isbuy):
        return self.params.stake
```

这非常简单，因为 Sizer 不进行计算，只是直接返回参数。

但该机制应该能够构建复杂的头寸系统来管理进出市场时的头寸。

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
- 根据 `position.size` 决定是否加倍固定头寸。
- 返回计算的值。

这样策略就不需要决定是否反转头寸或开仓，Sizer 负责控制，可以随时替换而不影响策略逻辑。

## 实用 Sizer 应用

不考虑复杂的头寸算法，可以使用两种不同的 Sizer 将策略从仅做多转换为多空策略。只需在 `cerebro` 执行中更改 Sizer，策略的行为就会发生变化。一个非常简单的收盘价交叉 SMA 策略：

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

注意策略并没有考虑当前头寸（通过 `self.position` 检查）来决定是否执行买入或卖出操作。只有交叉信号被考虑，Sizer 将负责所有事务。

这个 Sizer 仅在卖出时（如果已有头寸）返回非零头寸大小：

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

将所有内容放在一起（假设已导入 backtrader 并向系统添加了数据）：

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
- 现金水平从未恢复到初始值，因为策略始终在市场内。

两种方法都有负面效果，但这只是一个示例。

## bt.Sizer 参考

```python
class backtrader.Sizer()
```

这是 Sizer 的基类。任何 Sizer 都应继承此类并覆盖 `_getsizing` 方法。

**成员属性**：

- `strategy`：由 Sizer 所在策略设置。可以访问策略的整个 API，例如获取实际数据头寸：

```python
position = self.strategy.getposition(data)
```

- `broker`：由 Sizer 所在策略设置。复杂 Sizer 可能需要的信息（如投资组合价值等）可以通过它访问。

```python
def _getsizing(comminfo, cash, data, isbuy)
```

此方法必须由 Sizer 的子类覆盖以提供数量计算功能。

**参数**：

- `comminfo`：包含数据佣金信息的 `CommissionInfo` 实例，用于计算头寸价值、操作成本和操作佣金。
- `cash`：经纪商的当前可用现金。
- `data`：操作的目标。
- `isbuy`：买入操作为 `True`，卖出操作为 `False`。

此方法必须返回要执行的实际数量（整数）。如果返回 0，则不执行任何操作。

返回值的绝对值将被使用。
