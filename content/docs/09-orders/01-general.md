---
title: "订单"
weight: 1
---

# 订单

{{< youtube nvim FRFs28J14Gs >}}

Cerebro 是 backtrader 的核心控制系统，而 Strategy（由用户子类化）是用户的主要操作入口。策略需要连接系统的其他部分，订单正是为此而生。

订单将策略的逻辑决策转化为 Broker 可执行的消息，通过以下方式完成：

## 创建

通过 Strategy 的方法：`buy`、`sell` 和 `close`（Strategy），这些方法返回一个订单实例作为参考。

## 取消

通过 Strategy 的方法：`cancel`（Strategy），该方法需要一个订单实例来操作。

订单也用于向用户反馈 Broker 中的执行情况。

## 通知

通过 Strategy 的方法：`notify_order`（Strategy），该方法报告一个订单实例。

## 订单创建

调用 `buy`、`sell` 和 `close` 时，可用以下参数：

参数名          | 默认值         | 描述
--------------- | -------------- | -------------------
`data`          | None           | 为哪个数据创建订单。如果为 None，则使用系统中的第一个数据，`self.datas[0]` 或 `self.data0`（又名 `self.data`）。
`size`          | None           | 使用的单位数量。如果为 None，则使用通过 `getsizer` 获取的 sizer 实例来确定大小。
`price`         | None           | 使用的价格（实时 Broker 可能会对格式有实际限制，如果不符合最小刻度要求）。对于 Market 和 Close 订单，None 是有效的（市场决定价格）。对于 Limit、Stop 和 StopLimit 订单，该值决定触发点（在 Limit 的情况下，触发点显然是订单匹配的价格）。
`plimit`        | None           | 仅适用于 StopLimit 订单。这是在 Stop 触发后设置隐含 Limit 订单的价格。
`exectype`      | None           | 可能的值：<ul style="list-style-type: none;padding-left: 0; margin-left: 0;"><li>- `Order.Market` 或 None：市场订单将以下一个可用价格执行。在回测中，这将是下一根K线的开盘价。</li><li>- `Order.Limit`：只能在给定价格或更好的价格执行的订单。</li><li>- `Order.Stop`：在价格触发时执行的订单，执行方式如同 Market 订单。</li><li>- `Order.StopLimit`：在价格触发时执行的订单，作为隐含 Limit 订单执行，价格由 `pricelimit` 给定。</li></li>
`valid`         | None           | 可能的值：<ul style="list-style-type: none;padding-left: 0; margin-left: 0;"><li>- None：生成一个不会过期的订单（即 Good till cancel），并在市场中保留直到匹配或取消。实际上，Broker 往往会强制一个时间限制，但这通常是很长时间，所以认为它不会过期。</li><li>- `datetime.datetime` 或 `datetime.date` 实例：使用给定的日期生成一个有效直到该日期的订单（即 Good till date）。</li><li>- `Order.DAY` 或 0 或 `timedelta()`：生成一个有效期为一天的订单（即日订单），有效期直到会话结束。</li><li>- 数值：假定为一个对应于 `matplotlib` 编码的日期时间值（backtrader 使用的），并用于生成一个有效期至该时间的订单（即 Good till date）。</li></ul>
`tradeid`       | 0              | 这是 backtrader 用来跟踪同一资产上重叠交易的内部值。在通知订单状态变化时，该 `tradeid` 会返回给策略。
`**kwargs`      | /              | 额外的 Broker 实现可能支持额外的参数。backtrader 会将 kwargs 传递给创建的订单对象。

## 示例

如果 backtrader 默认的 4 种订单执行类型不够用，例如对接 Interactive Brokers 时，可以传递如下参数：

```python
orderType='LIT', lmtPrice=10.0, auxPrice=9.8
```

这将覆盖 backtrader 的设置，生成一个触及价格为 9.8 且限价为 10.0 的 LIMIT IF TOUCHED 订单。

**注意：**

`close` 方法会检查当前仓位，自动使用 `buy` 或 `sell` 平仓。如果不传参数，大小会自动计算；传入参数则可实现部分平仓或反向开仓。

## 订单通知

要接收通知，需要在自定义的 Strategy 子类中重写 `notify_order` 方法（默认什么都不做）。通知规则如下：

- 在策略的 `next` 方法之前发出。
- 在同一个 `next` 循环中，同一订单可能（且通常会）多次通知不同状态。
- 订单可能被提交、接受并在下一次 `next` 调用前完成执行。

这种情况下至少会有 3 次通知，状态依次为：

- `Order.Submitted`：订单已发送给 Broker。
- `Order.Accepted`：Broker 已接受订单，等待执行。
- `Order.Completed`：订单已快速匹配并完全成交（常见于 Market 订单）。

`Order.Partial` 状态在回测 Broker 中不会出现（不考虑部分成交），但在实际 Broker 中很常见。

实际 Broker 可能在更新仓位前发出一次或多次执行通知，这些执行构成一个 `Order.Partial` 通知。

实际执行数据保存在 `order.executed` 属性中，这是一个 `OrderData` 类型对象，包含大小、价格等常见字段。

创建时的值保存在 `order.created` 中，在整个订单生命周期内保持不变。

## 订单状态值

以下定义：

- `Order.Created`：创建订单实例时设置。除非手动创建，否则用户不会看到此状态。
- `Order.Submitted`：订单已发送给 Broker。回测模式会立即触发，但实际 Broker 可能需要时间，可能在转发给交易所时才通知。
- `Order.Accepted`：Broker 已接受订单，正在系统中（或交易所）等待执行，包含执行类型、大小、价格和有效期等参数。
- `Order.Partial`：订单已部分执行。`order.executed` 包含当前成交的大小和平均价格。
  - `order.executed.exbits` 包含部分成交的完整执行位列表。
- `Order.Complete`：订单已完全成交。
- `Order.Rejected`：Broker 拒绝了订单，可能因为某个参数（如有效期）不被接受。
  - 原因通过策略的 `notify_store` 方法通知。实际 Broker 会通过事件通知，可能与订单直接相关也可能不相关。
  - 回测 Broker 中不会出现此状态。
- `Order.Margin`：订单执行会导致保证金不足，之前接受的订单已从系统中移除。
- `Order.Cancelled`（或 `Order.Canceled`）：用户取消请求的确认。
  - 注意，调用 `cancel` 方法并不保证订单一定会被取消。订单可能已执行，但 Broker 尚未通知策略。
- `Order.Expired`：之前接受的订单已过期，已被从系统中移除。

## 引用：Order 及相关类
这些是 backtrader 生态系统中的通用类。与不同 Broker 集成时，它们可能被扩展或包含额外信息，请参阅相应 Broker 的文档。

```python
class backtrader.order.Order()
```

包含创建和执行数据以及订单类型的类。

订单可能具有以下状态：

- `Submitted`：发送给 Broker，等待确认。
- `Accepted`：被 Broker 接受。
- `Partial`：部分执行。
- `Completed`：完全执行。
- `Canceled/Cancelled`：被用户取消。
- `Expired`：过期。
- `Margin`：没有足够的现金执行订单。
- `Rejected`：被 Broker 拒绝。
  - 可能发生在订单提交期间（因此不会达到 `Accepted` 状态），或执行前因价格变动导致现金被其他用途（如期货保证金）占用。

成员属性：

- `ref`：唯一订单标识符。
- `created`：持有创建数据的 `OrderData`。
- `executed`：持有执行数据的 `OrderData`。
- `info`：通过 `addinfo()` 方法传递的自定义信息。以 `OrderedDict` 子类形式保存，键也可用 `.` 符号访问。

用户方法：

- `isbuy()`：返回布尔值，判断是否为买入订单。
- `issell()`：返回布尔值，判断是否为卖出订单。
- `alive()`：如果订单状态为 `Partial` 或 `Accepted`，返回 `True`。

```python
class backtrader.order.OrderData(dt=None, size=0, price=0.0, pricelimit=0.0, remsize

=0, pclose=0.0, trailamount=0.0, trailpercent=0.0)
```

包含订单创建和执行的实际数据。

创建时保存的是请求参数，执行时保存的是实际结果。

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
