---
title: "Position"
weight: 5
---

# Position

通常在策略中要检查资产 Position，或成为仓位、头寸，通过 `position` 属性或 `getposition(data=None, broker=None)` 方法。这将返回策略在cerebro提供的默认经纪商中的`datas[0]`的头寸。

头寸只是一个表示：

- 持有的资产数量（size）
- 平均价格（price）

它用作状态指示，例如可以用于决定是否需要发出订单（例如：仅在没有持仓时进入多头头寸）。

## 参考

```python
class backtrader.position.Position(size=0, price=0.0)
```

保存并更新头寸的数量和价格。该对象与任何资产没有关系。它只保存数量和价格。

成员属性：

- `size`（int）：当前头寸的数量
- `price`（float）：当前头寸的价格

可以使用 `len(position)` 来测试头寸实例以查看数量是否不为零。
