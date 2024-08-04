---
title: "用户定义的佣金"
weight: 3
---

### 用户定义的佣金

重新设计 `CommInfo` 对象以实现当前版本的最重要部分包括：

- 保留原始 `CommissionInfo` 类和行为
- 为轻松创建用户定义的佣金打开大门
- 将格式 `xx%` 作为新佣金方案的默认值，而不是 `0.xx`（这只是个口味问题），同时保持行为可配置

#### 定义佣金方案

这涉及 1 到 2 个步骤：

1. 子类化 `CommInfoBase`

   仅更改默认参数可能就足够了。`backtrader` 已经在模块 `backtrader.commissions` 中使用一些定义进行了此操作。期货的常规行业标准是每合同和每轮固定金额。定义可以这样做：

   ```python
   class CommInfo_Futures_Fixed(CommInfoBase):
       params = (
           ('stocklike', False),
           ('commtype', CommInfoBase.COMM_FIXED),
       )
   ```

   对于股票和按百分比计算的佣金：

   ```python
   class CommInfo_Stocks_Perc(CommInfoBase):
       params = (
           ('stocklike', True),
           ('commtype', CommInfoBase.COMM_PERC),
       )
   ```

   如上所述，此处解释百分比的默认值（作为参数 `commission` 传递）为：`xx%`。如果需要旧的/其他行为 `0.xx`，可以轻松实现：

   ```python
   class CommInfo_Stocks_PercAbs(CommInfoBase):
       params = (
           ('stocklike', True),
           ('commtype', CommInfoBase.COMM_PERC),
           ('percabs', True),
       )
   ```

2.（如有必要）重写 `_getcommission` 方法

   定义如下：

   ```python
   def _getcommission(self, size, price, pseudoexec):
       '''Calculates the commission of an operation at a given price

       pseudoexec: if True the operation has not yet been executed
       '''
   ```

   下面的实际示例中有更多详细信息。

#### 如何将其应用于平台

一旦有了 `CommInfoBase` 子类，诀窍是使用 `broker.addcommissioninfo` 而不是通常的 `broker.setcommission`。后者将在内部使用旧版 `CommissionInfoObject`。

```python
...

comminfo = CommInfo_Stocks_PercAbs(commission=0.005)  # 0.5%
cerebro.broker.addcommissioninfo(comminfo)
```

`addcommissioninfo` 方法定义如下：

```python
def addcommissioninfo(self, comminfo, name=None):
    self.comminfo[name] = comminfo
```

设置 `name` 意味着 `comminfo` 对象将仅适用于具有该名称的资产。默认值 `None` 表示它适用于系统中的所有资产。

#### 实际示例

问题 #45 询问一种适用于期货的佣金方案，该方案基于百分比，并使用整个“虚拟”合约价值的佣金百分比，即：包括期货乘数在佣金计算中。

```python
import backtrader as bt

class CommInfo_Fut_Perc_Mult(bt.CommInfoBase):
    params = (
      ('stocklike', False),  # Futures
      ('commtype', bt.CommInfoBase.COMM_PERC),  # Apply % Commission
    # ('percabs', False),  # pass perc as xx% which is the default
    )

    def _getcommission(self, size, price, pseudoexec):
        return size * price * self.p.commission * self.p.mult
```

将其放入系统中：

```python
comminfo = CommInfo_Fut_Perc_Mult(
    commission=0.1,  # 0.1%
    mult=10,
    margin=2000  # Margin is needed for futures-like instruments
)

cerebro.addcommissioninfo(comminfo)
```

如果更喜欢 `0.xx` 格式作为默认值，只需将参数 `percabs` 设置为 `True`：

```python
class CommInfo_Fut_Perc_Mult(bt.CommInfoBase):
    params = (
      ('stocklike', False),  # Futures
      ('commtype', bt.CommInfoBase.COMM_PERC),  # Apply % Commission
      ('percabs', True),  # pass perc as 0.xx
    )

comminfo = CommInfo_Fut_Perc_Mult(
    commission=0.001,  # 0.1%
    mult=10,
    margin=2000  # Margin is needed for futures-like instruments
)

cerebro.addcommissioninfo(comminfo)
```

#### 解释 `pseudoexec`

让我们回顾 `_getcommission` 的定义：

```python
def _getcommission(self, size, price, pseudoexec):
    '''Calculates the commission of an operation at a given price

    pseudoexec: if True the operation has not yet been executed
    '''
```

`pseudoexec` 参数的目的是在平台可能调用此方法进行可用现金的预计算和其他一些任务时使用。

这意味着该方法可能（实际上确实会）使用相同的参数多次调用。

`pseudoexec` 指示调用是否对应于订单的实际执行。尽管乍看之下这似乎不“相关”，但在以下情景中却是如此：

经纪商在期货往返佣金超过 5000 单位合同时提供 50% 的折扣。

在这种情况下，如果没有 `pseudoexec`，对该方法的多次非执行调用将迅速触发折扣到位的假设。

将此情景付诸实践：

```python
import backtrader as bt

class CommInfo_Fut_Discount(bt.CommInfoBase):
    params = (
      ('stocklike', False),  # Futures
      ('commtype', bt.CommInfoBase.COMM_FIXED),  # Apply Commission

      # Custom params for the discount
      ('discount_volume', 5000),  # minimum contracts to achieve discount
      ('discount_perc', 50.0),  # 50.0% discount
    )

    negotiated_volume = 0  # attribute to keep track of the actual volume

    def _getcommission(self, size, price, pseudoexec):
        if self.negotiated_volume > self.p.discount_volume:
           actual_discount = self.p.discount_perc / 100.0
        else:
           actual_discount = 0.0

        commission = self.p.commission * (1.0 - actual_discount)
        commvalue = size * price * commission

        if not pseudoexec:
           # keep track of actual real executed size for future discounts
           self.negotiated_volume += size

        return commvalue
```

#### `CommInfoBase` 文档字符串和参数

有关 `CommInfoBase` 的参考，请参见“佣金：股票与期货”。
