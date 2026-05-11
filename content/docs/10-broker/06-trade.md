---
title: "Trade"
weight: 6
---

# Trade

交易的定义：当某个工具的头寸从 0 变为 X（正数为多头，负数为空头）时，交易开启；当头寸从 X 变为 0 时，交易关闭。

以下两种情况：

- 从正变负
- 从负变正

实际上被视为：关闭一个交易（头寸从 X 到 0），同时开启一个新交易（头寸从 0 到 Y）。

交易仅用于信息展示，用户无法调用其方法。

## 参考

```python
class backtrader.trade.Trade(data=None, tradeid=0, historyon=False, size=0, price=0.0, value=0.0, commission=0.0)
```

追踪交易的生命周期：数量、价格、佣金和价值。交易从 0 开始，可以增加和减少，回到 0 表示关闭。交易可以是多头（正数）或空头（负数）。交易不支持反转。

成员属性          | 描述
----------------- | ---------------------
`ref`             | 唯一的交易标识符
`status`          | `Created`, `Open`, `Closed`之一
`tradeid`         | 在创建订单时传递给订单的分组交易ID，订单的默认值为0
`size`            | 当前交易的数量
`price`           | 当前交易的价格
`value`           | 当前交易的价值
`commission`      | 当前累计的佣金
`pnl`             | 当前交易的盈亏（毛利）
`pnlcomm`         | 当前交易的净盈亏（扣除佣金后的净利润）
`isclosed`        | 记录最后一次更新是否关闭了交易（将交易数量设为零）
`isopen`          | 记录任何更新是否开启了交易
`justopened`      | 如果交易刚刚开启
`baropen`         | 交易开启的bar
`dtopen`          | 交易开启的浮点编码日期时间，使用 `open_datetime` 方法获取 Python `datetime.datetime` 或使用平台提供的 `num2date` 方法
`barclose`        | 交易关闭的bar
`dtclose`         | 交易关闭的浮点编码日期时间，使用 `close_datetime` 方法获取 Python `datetime.datetime` 或使用平台提供的 `num2date` 方法
`barlen`          | 交易开启的bar数量
`historyon`       | 是否记录历史
`history`         | 包含每次“更新”事件更新后的状态和使用的参数的列表，历史记录的第一个条目是开启事件，最后一个条目是关闭事件
