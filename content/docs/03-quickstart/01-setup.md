---
title: "环境设置"
weight: 1
---

# 初始配置

{{< youtube Li6h94v3zYg >}}

在写复杂策略前，先搭好运行环境很重要。Backtrader 的核心是 Cerebro，它负责管理数据、策略和资金。

如下是最简单的初始配置代码：

```python
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()

    print(f'初始资金: {cerebro.broker.getvalue()}')
    cerebro.run()
    print(f'最终资金: {cerebro.broker.getvalue()}')
```

先导入 backtrader 模块并命名为 bt，然后基于 `bt.Cerebro` 实例化引擎。

```python
import backtrader as bt

cerebro = bt.Cerebro()
```

通过 `cerebro.broker.getvalue()` 获取并打印初始组合价值。接着运行 `cerebro.run()` 迭代数据进行模拟交易，完成后再次打印最终的持仓组合价值。

```python
cerebro.run()
print(f'最终资金: {cerebro.broker.getvalue()}')
```

输出如下：

```
初始资金: 10000.00
最终资金: 10000.00
```

这个简单示例中，Cerebro 引擎在后台自动创建了 broker 实例并分配了初始资金。如未明确设置 broker，系统会使用默认配置，初始资金为 10,000 货币单位。

接下来的章节会对 broker 进行更多配置，逐步添加数据源（DataFeed）、策略（Strategy）、指标（Indicator）等，完善这个交易系统。
