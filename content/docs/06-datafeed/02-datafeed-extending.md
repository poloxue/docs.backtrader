---
title: "扩展数据源"
weight: 2
--- 

# 扩展数据源

在 GitHub 上的问题实际上推动了文档部分的完成，或帮助我理解 backtrader 是否具有我从一开始设想的易用性和灵活性，以及在此过程中做出的决策。

在这种情况下是 Issue #9。

问题似乎最终归结为：

用户是否可以轻松扩展现有机制，以添加额外的信息（例如行），并将其与现有的价格信息（如开盘价、高价等）一起传递？
据我了解，答案是：可以。

发布者似乎有这些要求（来自 Issue #6）：

- 一个数据源，正在解析为 CSV 格式
- 使用 `GenericCSVData` 加载信息
- 这种通用 CSV 支持是为了响应 Issue #6 开发的
- 一个额外的字段，显然包含 P/E 信息，需要与解析的 CSV 数据一起传递

让我们基于 CSV 数据源开发和 `GenericCSVData` 示例帖子构建。

#### 步骤：

1. 假设 P/E 信息已设置在被解析的 CSV 数据中
2. 使用 `GenericCSVData` 作为基类
3. 使用 `pe` 扩展现有的行（开盘价/最高价/最低价/收盘价/成交量/持仓兴趣）
4. 添加一个参数，让调用者确定 P/E 信息的列位置

结果如下：

```python
from backtrader.feeds import GenericCSVData

class GenericCSV_PE(GenericCSVData):

    # 添加 'pe' 行到从基类继承的行中
    lines = ('pe',)

    # GenericCSVData 中的 openinterest 索引为 7 ... 添加 1
    # 将参数添加到从基类继承的参数中
    params = (('pe', 8),)
```

这样工作就完成了...

稍后在策略中使用此数据源时：

```python
import backtrader as bt

....

class MyStrategy(bt.Strategy):

    ...

    def next(self):

        if self.data.close > 2000 and self.data.pe < 12:
            # TORA TORA TORA --- 退出市场
            self.sell(stake=1000000, price=0.01, exectype=Order.Limit)
    ...
```

#### 绘制额外的 P/E 行

显然，数据源中没有自动绘制支持这个额外的行。

最好的替代方法是在该行上进行简单移动平均并在单独的轴上绘制：

```python
import backtrader as bt
import backtrader.indicators as btind

....

class MyStrategy(bt.Strategy):

    def __init__(self):

        # 指标自动注册，即使在类中没有保留明显的引用也会绘制
        btind.SMA(self.data.pe, period=1, subplot=False)

    ...

    def next(self):

        if self.data.close > 2000 and self.data.pe < 12:
            # TORA TORA TORA --- 退出市场
            self.sell(stake=1000000, price=0.01, exectype=Order.Limit)
    ...
```
