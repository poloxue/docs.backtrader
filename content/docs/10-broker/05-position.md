---
title: "Position"
weight: 5
---

# Position

在策略中检查资产的头寸（Position），可通过 `position` 属性或 `getposition(data=None, broker=None)` 方法。这会返回策略在 Cerebro 默认 Broker 中 `datas[0]` 的头寸。

头寸仅表示：

- 持有数量（size）
- 平均价格（price）

它用作状态指示，例如决定是否需要发出订单（如仅在没有持仓时开仓）。

## 参考

```python
class backtrader.position.Position(size=0, price=0.0)
```

保存并更新头寸的数量和价格。该对象与任何资产没有关系。它只保存数量和价格。

成员属性：

- `size`（int）：当前头寸的数量
- `price`（float）：当前头寸的价格

可以使用 `len(position)` 来测试头寸实例以查看数量是否不为零。
