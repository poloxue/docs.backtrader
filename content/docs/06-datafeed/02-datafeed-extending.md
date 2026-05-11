---
title: "扩展数据源"
weight: 2
--- 

# 扩展数据源

能否轻松扩展现有机制，添加额外字段并与价格数据（如开盘价、最高价等）一起传递？

- 正在解析一个 CSV 格式的数据源
- 使用 `GenericCSVData` 加载数据
- 通用 CSV 支持是为响应 Issue #6 开发的
- 需要将 P/E 字段与解析的 CSV 数据一起传递

下面基于 CSV 数据源开发和 `GenericCSVData` 示例来演示。

### 步骤：

1. 假设 P/E 信息已包含在 CSV 数据中
2. 使用 `GenericCSVData` 作为基类
3. 添加 `pe` 行扩展已有的字段（开盘价/最高价/最低价/收盘价/成交量/持仓兴趣）
4. 添加参数，让调用者指定 P/E 列的索引

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

这样就完成了。

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

### 绘制额外的 P/E 行

数据源中没有自动绘制此行数据的支持。

替代方案是在该行上添加简单移动平均线，在单独的子图上绘制：

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
