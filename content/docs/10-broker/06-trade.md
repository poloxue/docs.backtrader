---
title: "交易 Trade"
weight: 6
---

交易

交易的定义：

当一个工具的头寸从0变为一个大小X（多头/空头头寸分别为正/负）时，交易被认为是开启的。

当一个头寸从X变为0时，交易被认为是关闭的。

以下两种操作：

从正变负

从负变正

实际上被视为：

一个交易被关闭了（头寸从X变为0）

一个新的交易被开启了（头寸从0变为Y）

交易仅用于信息展示，用户无法调用其方法。

参考：交易

```python
class backtrader.trade.Trade(data=None, tradeid=0, historyon=False, size=0, price=0.0, value=0.0, commission=0.0)
```

追踪交易的生命周期：数量、价格、佣金（和价值？）

一个交易从0开始，可以增加和减少，如果回到0可以被认为是关闭的。

交易可以是多头（正数）或空头（负数）

交易不支持反转（逻辑中不支持）

成员属性：

- `ref`：唯一的交易标识符
- `status`（int）：`Created`, `Open`, `Closed`之一
- `tradeid`：在创建订单时传递给订单的分组交易ID，订单的默认值为0
- `size`（int）：当前交易的数量
- `price`（float）：当前交易的价格
- `value`（float）：当前交易的价值
- `commission`（float）：当前累计的佣金
- `pnl`（float）：当前交易的盈亏（毛利）
- `pnlcomm`（float）：当前交易的净盈亏（扣除佣金后的净利润）
- `isclosed`（bool）：记录最后一次更新是否关闭了交易（将交易数量设为零）
- `isopen`（bool）：记录任何更新是否开启了交易
- `justopened`（bool）：如果交易刚刚开启
- `baropen`（int）：交易开启的bar
- `dtopen`（float）：交易开启的浮点编码日期时间

  使用 `open_datetime` 方法获取 Python `datetime.datetime` 或使用平台提供的 `num2date` 方法
- `barclose`（int）：交易关闭的bar
- `dtclose`（float）：交易关闭的浮点编码日期时间

  使用 `close_datetime` 方法获取 Python `datetime.datetime` 或使用平台提供的 `num2date` 方法
- `barlen`（int）：交易开启的bar数量
- `historyon`（bool）：是否记录历史
- `history`（list）：包含每次“更新”事件更新后的状态和使用的参数的列表

  历史记录的第一个条目是开启事件，最后一个条目是关闭事件
