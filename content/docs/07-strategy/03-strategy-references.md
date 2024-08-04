---
title: "策略参考"
weight: 3
---

### 策略参考

#### 内置策略参考

#### MA_CrossOver
别名：
- SMA_CrossOver

这是一个仅做多的策略，基于移动平均线交叉操作。

**注意：**

- 尽管默认...

**买入逻辑：**

- 数据上没有头寸。
- `fast` 移动平均线向上穿过 `slow` 移动平均线。

**卖出逻辑：**

- 数据上存在头寸。
- `fast` 移动平均线向下穿过 `slow` 移动平均线。

**订单执行类型：**

- 市价单

**线：**

- datetime

**参数：**

- fast (10)
- slow (30)
- _movav (<class ‘backtrader.indicators.sma.SMA’>)

#### SignalStrategy
此策略的子类旨在使用信号自动操作。

信号通常是指标，预期输出值为：

- > 0 表示多头指示
- < 0 表示空头指示

信号分为两组，共有 5 种类型。

**主要组：**

- `LONGSHORT`：接受来自该信号的多头和空头指示。
- `LONG`：
  - 接受多头指示进行做多。
  - 接受空头指示平仓多头。但：
    - 如果系统中有 `LONGEXIT` 信号，将用它来平仓多头。
    - 如果有 `SHORT` 信号且没有 `LONGEXIT` 信号，它将被用来平仓多头再开空头。
- `SHORT`：
  - 接受空头指示进行做空。
  - 接受多头指示平仓空头。但：
    - 如果系统中有 `SHORTEXIT` 信号，将用它来平仓空头。
    - 如果有 `LONG` 信号且没有 `SHORTEXIT` 信号，它将被用来平仓空头再开多头。

**退出组：**

这两个信号旨在覆盖其他信号，并为平仓提供标准。

- `LONGEXIT`：接受空头指示平仓多头。
- `SHORTEXIT`：接受多头指示平仓空头。

**订单发出：**

- 订单执行类型为市价单，有效期为“直到取消” (Good until Canceled)。

**参数：**

- signals (默认值: []): 允许实例化信号并分配到正确类型的列表/元组。
- _accumulate (默认值: False): 允许进入市场（多头/空头），即使已经在市场中。
- _concurrent (默认值: False): 允许在已有待执行订单时发出新订单。
- _data (默认值: None): 如果系统中存在多个数据，目标数据是哪一个。这可以是：
  - None: 将使用系统中的第一个数据。
  - int: 表示在该位置插入的数据。
  - str: 创建数据时给定的名称（参数 name），或通过 cerebro.adddata(..., name=) 添加时给定的名称。
  - 数据实例。

**线：**

- datetime

**参数：**

- signals ([])
- _accumulate (False)
- _concurrent (False)
- _data (None)
