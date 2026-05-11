---
title: "自定义佣金"
weight: 3
---

# 自定义佣金

当前版本重新设计 `CommInfo` 对象，核心目标包括：

- 保留原始 `CommissionInfo` 类和行为
- 为用户轻松创建自定义佣金方案提供支持
- 将 `xx%` 格式作为新佣金方案的默认值，而不是 `0.xx`（这只是个人偏好），同时保持行为可配置

## 定义佣金方案

这需要 1 到 2 个步骤：

### 子类化 `CommInfoBase`

仅更改默认参数可能就足够了。`backtrader` 在 `backtrader.commissions` 模块中已经有一些预定义。期货的行业标准是每份合约固定金额。可以这样定义：

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

   如上所述，百分比的默认值（通过参数 `commission` 传递）为 `xx%`。如果需要旧的行为 `0.xx`，可以这样实现：

   ```python
   class CommInfo_Stocks_PercAbs(CommInfoBase):
       params = (
           ('stocklike', True),
           ('commtype', CommInfoBase.COMM_PERC),
           ('percabs', True),
       )
   ```

### 重写 `_getcommission` 方法（如有必要）

定义如下：

```python
def _getcommission(self, size, price, pseudoexec):
   '''Calculates the commission of an operation at a given price

   pseudoexec: if True the operation has not yet been executed
   '''
```

   下面的实际示例中有更详细的说明。

#### 如何将其应用于平台

有了 `CommInfoBase` 子类后，需要使用 `broker.addcommissioninfo` 而不是常用的 `broker.setcommission`。后者会在内部使用旧版 `CommissionInfoObject`。

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

设置 `name` 后，`comminfo` 对象将仅适用于具有该名称的资产。默认值 `None` 表示适用于系统中的所有资产。

## 实际示例

问题 #45 提出了一种适用于期货的佣金方案，基于百分比，并使用整个”虚拟”合约价值的佣金百分比，即在佣金计算中包括期货乘数。

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

如果更倾向于 `0.xx` 格式，只需将参数 `percabs` 设置为 `True`：

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

## 解释 `pseudoexec`

让我们回顾 `_getcommission` 的定义：

```python
def _getcommission(self, size, price, pseudoexec):
    '''Calculates the commission of an operation at a given price

    pseudoexec: if True the operation has not yet been executed
    '''
```

`pseudoexec` 参数用于平台可能调用此方法进行可用现金预计算等任务。

这意味着该方法可能（实际上也确实会）使用相同参数被多次调用。

`pseudoexec` 指示调用是否对应订单的实际执行。虽然乍看之下似乎无关紧要，但在以下场景中却很重要：

经纪商对期货交易量超过 5000 张合约时提供 50% 的佣金折扣。

在这种情况下，如果没有 `pseudoexec`，多次非执行调用会错误地触发折扣条件。

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

## `CommInfoBase` 文档字符串和参数

有关 `CommInfoBase` 的参考，请参见”佣金”一章。
