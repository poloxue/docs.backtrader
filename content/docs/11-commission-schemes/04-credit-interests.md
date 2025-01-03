---
title: "利息"
weight: 4
---

# 利息

在某些情况下，由于资产操作包含利率，实际经纪商的现金金额可能会减少。例如：

- 股票的卖空
- ETF 的多头和空头操作

该费用直接从经纪账户的现金余额中扣除。但它仍然可以看作是佣金方案的一部分。因此，它已在 `backtrader` 中进行了建模。

`CommInfoBase` 类（以及主要的 `CommissionInfo` 接口对象）已扩展了：

- 两个新参数，用于设置利率并确定是否仅应用于空头还是同时应用于多头和空头

## 参数

- `interest`（默认值：0.0）

  如果非零，这是持有卖空头寸时收取的年度利息。这主要用于股票卖空。

  应用的默认公式：`days * price * size * (interest / 365)`

  必须以绝对值指定：0.05 -> 5%

  注意：可以通过重写方法 `get_credit_interest` 来更改行为

- `interest_long`（默认值：False）

  一些产品如 ETF，在多头和空头头寸上都收取利息。如果为 `True` 并且 `interest` 非零，则多头和空头方向都将收取利息。

## 公式

默认实现将使用以下公式：

```python
days * abs(size) * price * (interest / 365)
```

其中：

- `days`：自头寸开立或上次利息计算以来经过的天数

## 重写公式

要更改公式，需要子类化 `CommissionInfo`。需要重写的方法是：

```python
def _get_credit_interest(self, size, price, days, dt0, dt1):
    '''
    此方法返回经纪商收取的利息成本。

    对于 ``size > 0`` 的情况，仅在类参数 ``interest_long`` 为 ``True`` 时调用此方法。

    计算利率的公式为：

    公式：``days * price * abs(size) * (interest / 365)``

    参数：
      - ``data``：收取利息的数据源
      - ``size``：当前头寸大小。> 0 表示多头头寸，< 0 表示空头头寸（此参数不会为 ``0``）
      - ``price``：当前头寸价格
      - ``days``：自上次利息计算以来经过的天数（这是（dt0 - dt1）.days）
      - ``dt0``：当前日期时间（datetime.datetime）
      - ``dt1``：上次计算日期时间（datetime.datetime）

    ``dt0`` 和 ``dt1`` 在默认实现中未使用，并作为重写方法的额外输入提供
    '''
```

可能经纪商在计算利率时不考虑周末或银行假日。在这种情况下，子类如下：

```python
import backtrader as bt

class MyCommissionInfo(bt.CommInfo):

   def _get_credit_interest(self, size, price, days, dt0, dt1):
       return 1.0 * abs(size) * price * (self.p.interest / 365.0)
```

在这种情况下，公式中：

- `days` 被替换为 `1.0`
- 因为如果周末/银行假日不算数，则下一次计算总是在上一次计算后的一个交易日进行。
