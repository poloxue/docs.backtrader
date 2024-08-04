---
title: "订单"
weight: 1
---

# 订单

Cerebro 是 backtrader 中的关键控制系统，而 Strategy（一个子类）是终端用户的关键控制点。后者需要一个连接系统其他部分的方法，这就是订单发挥关键作用的地方。

订单将策略中的逻辑决策转化为适合 Broker 执行操作的消息。这是通过以下方式完成的：

### 创建

通过 Strategy 的方法：`buy`、`sell` 和 `close`（Strategy），这些方法返回一个订单实例作为参考。

### 取消

通过 Strategy 的方法：`cancel`（Strategy），该方法需要一个订单实例来操作。

订单也作为一种通信方式反馈给用户，通知 Broker 中的执行情况。

### 通知

通过 Strategy 的方法：`notify_order`（Strategy），该方法报告一个订单实例。

### 订单创建
调用 `buy`、`sell` 和 `close` 时，以下参数适用于创建：

- `data`（默认：None）

  为哪个数据创建订单。如果为 None，则使用系统中的第一个数据，`self.datas[0]` 或 `self.data0`（又名 `self.data`）。

- `size`（默认：None）

  使用的单位数量。如果为 None，则使用通过 `getsizer` 获取的 sizer 实例来确定大小。

- `price`（默认：None）

  使用的价格（实时 Broker 可能会对格式有实际限制，如果不符合最小刻度要求）。对于 Market 和 Close 订单，None 是有效的（市场决定价格）。对于 Limit、Stop 和 StopLimit 订单，该值决定触发点（在 Limit 的情况下，触发点显然是订单匹配的价格）。

- `plimit`（默认：None）

  仅适用于 StopLimit 订单。这是在 Stop 触发后设置隐含 Limit 订单的价格。

- `exectype`（默认：None）

  可能的值：
  - `Order.Market` 或 None：市场订单将以下一个可用价格执行。在回测中，这将是下一根K线的开盘价。
  - `Order.Limit`：只能在给定价格或更好的价格执行的订单。
  - `Order.Stop`：在价格触发时执行的订单，执行方式如同 Market 订单。
  - `Order.StopLimit`：在价格触发时执行的订单，作为隐含 Limit 订单执行，价格由 `pricelimit` 给定。

- `valid`（默认：None）

  可能的值：
  - None：生成一个不会过期的订单（即 Good till cancel），并在市场中保留直到匹配或取消。实际上，Broker 往往会强制一个时间限制，但这通常是很长时间，所以认为它不会过期。
  - `datetime.datetime` 或 `datetime.date` 实例：使用给定的日期生成一个有效直到该日期的订单（即 Good till date）。
  - `Order.DAY` 或 0 或 `timedelta()`：生成一个有效期为一天的订单（即日订单），有效期直到会话结束。
  - 数值：假定为一个对应于 `matplotlib` 编码的日期时间值（backtrader 使用的），并用于生成一个有效期至该时间的订单（即 Good till date）。

- `tradeid`（默认：0）

  这是 backtrader 用来跟踪同一资产上重叠交易的内部值。在通知订单状态变化时，该 `tradeid` 会返回给策略。

- **kwargs：额外的 Broker 实现可能支持额外的参数。backtrader 会将 kwargs 传递给创建的订单对象。

示例：如果 backtrader 直接支持的 4 种订单执行类型不够，例如对于 Interactive Brokers，可以传递如下参数：

```python
orderType='LIT', lmtPrice=10.0, auxPrice=9.8
```

这将覆盖 backtrader 的设置，生成一个触及价格为 9.8 且限价为 10.0 的 LIMIT IF TOUCHED 订单。

**注意：**

`close` 方法将检查当前仓位，并相应地使用 `buy` 或 `sell` 有效地关闭仓位。除非参数由用户输入，否则大小也将自动计算，在这种情况下可以实现部分关闭或反转。

### 订单通知
要接收通知，必须在用户子类的 Strategy 中重写 `notify_order` 方法（默认行为是什么都不做）。以下适用于这些通知：

- 在策略的 `next` 方法调用之前发出。
- 在同一个 `next` 循环中，可能（并且会）多次发生相同或不同状态的相同订单通知。
- 订单可能会被提交给 Broker，被接受并在 `next` 再次调用之前完成执行。

在这种情况下，至少会发生 3 次通知，状态值如下：

- `Order.Submitted` 因为订单已发送给 Broker。
- `Order.Accepted` 因为订单已被 Broker 接受并等待执行。
- `Order.Completed` 因为在示例中订单已快速匹配并完全成交（通常适用于 Market 订单）。

对于 `Order.Partial` 状态，即使在回测 Broker 中（不考虑匹配量）不会看到，但在实际 Broker 中肯定会看到。

实际 Broker 可能会在更新仓位之前发出一次或多次执行通知，这些执行将构成一个 `Order.Partial` 通知。

实际执行数据在属性：`order.executed` 中，这是一个 `OrderData` 类型的对象，具有常见的字段如大小和价格。

创建时的值存储在 `order.created` 中，在订单生命周期中保持不变。

### 订单状态值
以下定义：

- `Order.Created`：在创建订单实例时设置。除非手动创建订单实例，否则终端用户不会看到该状态。
- `Order.Submitted`：在订单实例已传输给 Broker 时设置。这只是意味着已发送。在回测模式中这是立即的，但在实际 Broker 中可能需要时间，可能在转发给交易所时才通知。
- `Order.Accepted`：Broker 已接受订单，并在系统中（或已在交易所）等待执行，参数如执行类型、大小、价格和有效期。
- `Order.Partial`：订单已部分执行。`order.executed` 包含当前填充的大小和平均价格。
  - `order.executed.exbits` 包含部分填充的完整执行位列表。
- `Order.Complete`：订单已完全填充平均价格。
- `Order.Rejected`：Broker 拒绝了订单。可能是某个参数（如有效期）不被接受，订单无法接受。
  - 原因会通过策略的 `notify_store` 方法通知。虽然这可能显得奇怪，但实际 Broker 会通过事件通知，这可能与订单直接相关或无关。但可以在 `notify_store` 中看到 Broker 的通知。
  - 在回测 Broker 中不会看到此状态。
- `Order.Margin`：订单执行将导致保证金要求，之前接受的订单已从系统中移除。
- `Order.Cancelled`（或 `Order.Canceled`）：用户请求取消的确认。
  - 必须考虑，通过策略的 `cancel` 方法请求取消订单并不保证取消。订单可能已经执行，但此执行可能尚未被 Broker 通知，并且通知可能尚未传递给策略。
- `Order.Expired`：之前接受的订单具有时间有效性，已过期并被从系统中移除。

### 引用：Order 及相关类
这些对象是 backtrader 生态系统中的通用类。在与其他 Broker 操作时，它们可能已扩展和/或包含额外的嵌入信息。请参阅相应 Broker 的参考。

```python
class backtrader.order.Order()
```

持有创建/执行数据和订单类型的类。

订单可能具有以下状态：

- `Submitted`：发送给 Broker，等待确认。
- `Accepted`：被 Broker 接受。
- `Partial`：部分执行。
- `Completed`：完全执行。
- `Canceled/Cancelled`：被用户取消。
- `Expired`：过期。
- `Margin`：没有足够的现金执行订单。
- `Rejected`：被 Broker 拒绝。
  - 这可能发生在订单提交期间（因此订单不会达到 `Accepted` 状态），或者在执行前因每根新K线价格变化，现金被其他来源（如期货类工具）抽取导致拒绝。

成员属性：

- `ref`：唯一订单标识符。
- `created`：持有创建数据的 `OrderData`。
- `executed`：持有执行数据的 `OrderData`。
- `info`：通过 `addinfo()` 方法传递的自定义信息。以子类化的 `OrderedDict` 形式保存，键也可以使用`.`符号指定。

用户方法：

- `isbuy()`：返回一个布尔值，指示订单是否买入。
- `issell()`：返回一个布尔值，指示订单是否卖出。
- `alive()`：返回一个布尔值，如果订单状态为 `Partial` 或 `Accepted`。

```python
class backtrader.order.OrderData(dt=None, size=0, price=0.0, pricelimit=0.0, remsize

=0, pclose=0.0, trailamount=0.0, trailpercent=0.0)
```

持有创建和执行实际订单数据。

在创建情况下，请求的数据，在执行情况下，实际结果。

成员属性：

- `exbits`：此 `OrderData` 的 `OrderExecutionBits` 的可迭代对象。
- `dt`：日期时间（浮点数）创建/执行时间。
- `size`：请求/执行大小。
- `price`：执行价格。注意：如果没有价格和 `pricelimit`，订单创建时的收盘价将作为参考。
- `pricelimit`：StopLimit 的价格限制（首先有触发）。
- `trailamount`：追踪止损的绝对价格距离。
- `trailpercent`：追踪止损的百分比价格距离。
- `value`：整个位大小的市场价值。
- `comm`：整个位执行的佣金。
- `pnl`：此位产生的盈亏（如果有关闭）。
- `margin`：订单产生的保证金（如果有）。
- `psize`：当前未平仓头寸大小。
- `pprice`：当前未平仓头寸价格。

```python
class backtrader.order.OrderExecutionBit(dt=None, size=0, price=0.0, closed=0, closedvalue=0.0, closedcomm=0.0, opened=0, openedvalue=0.0, openedcomm=0.0, pnl=0.0, psize=0, pprice=0.0)
```

旨在持有订单执行信息。“位”不确定订单是否已完全/部分执行，只是持有信息。

成员属性：

- `dt`：日期时间（浮点数）执行时间。
- `size`：执行数量。
- `price`：执行价格。
- `closed`：执行关闭了多少现有头寸。
- `opened`：执行打开了多少新头寸。
- `openedvalue`：新开部分的市场价值。
- `closedvalue`：关闭部分的市场价值。
- `closedcomm`：关闭部分的佣金。
- `openedcomm`：打开部分的佣金。
- `value`：整个位大小的市场价值。
- `comm`：整个位执行的佣金。
- `pnl`：此位产生的盈亏（如果有关闭）。
- `psize`：当前未平仓头寸大小。
- `pprice`：当前未平仓头寸价格。
