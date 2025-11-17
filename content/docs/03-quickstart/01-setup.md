---
title: "环境设置"
weight: 1
---

# 初始配置

在写复杂策略前，先搭好运行环境很重要。Backtrader 的核心是 Cerebro，它负责管理数据、策略和资金，就像系统的大脑。

如下所示是最简单的初始配置代码：

```python
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()

    print(f'初始资金: {cerebro.broker.getvalue()}')
    cerebro.run()
    print(f'最终资金: {cerebro.broker.getvalue()})
```


首先导入了 backtrader 模块并命名为 bt。

```python
import backtrader as bt
```

并基于 bt.Cerebro 实例化了 Cerebro 引擎。

````python
cerebro = bt.Cerebro()
``````

通过 `cerebro.broker.getvalue()` 获取并打印了初始的组合价值。

```python
print(f'初始资金: {cerebro.broker.getvalue()}')
```

接着运行 `cerebro.run()` 迭代数据模拟交易。完成后，再次打印最终的持仓组合价值

```python
cerebro.run()
print(f'最终资金: {cerebro.broker.getvalue()})
```

输出如下：

```
初始资金: 10000.00
最终资金: 10000.00
```

这个基础设置是构建复杂交易策略的基础。

这一简单的示例中，Cerebro 引擎在后台创建了一个 broker 实例，并自动分配了一些初始资金。这种 broker 实例化是 backtrader 的常规特性，旨在简化用户操作。

如未明确设置 broker，系统会使用默认 broker，默认初始资金通常是 10,000 货币单位。

接下来的章节，会对 broker 进行更对的配置，继续添加数据源（DataFeed）、策略（Strategy）、指标（Indicator）等，逐步完善这个交易系统。
