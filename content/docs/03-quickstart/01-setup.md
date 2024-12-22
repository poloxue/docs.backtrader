---
title: "环境设置"
weight: 1
---

# 环境设置

在开始构建复杂的交易策略前，我们要先配置策略运行环境。Backtrader 的环境离不开一个核心类 Cerebro（大脑），后续会详细介绍它。

## 初始化配置

我们先看完整的环境初始化设置的代码： 

```python
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```


在这个示例中，我们首先导入了 backtrader 模块并命名为 bt。

```python
import backtrader as bt
```

并基于 bt.Cerebro 实例化了 Cerebro 引擎。

````python
cerebro = bt.Cerebro()
``````

我们通过 `cerebro.broker.getvalue()` 获取并打印了初始的持仓组合价值，即我们的初始资金。

```python
print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

接着运行 `cerebro.run()` 以处理数据模拟交易，并再次打印最终的持仓组合价值

```python
cerebro.run()
print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

输出如下：

```
Starting Portfolio Value: 10000.00
Final Portfolio Value: 10000.00
```

## 配置解析

这个基础设置是构建复杂交易策略的基础。

这一简单的示例中，Cerebro 引擎在后台创建了一个 broker 实例，并自动分配了一些初始资金。这种后台 broker 实例化是 backtrader 的常规特性，旨在简化用户操作。如果用户未明确设置 broker，系统会使用默认 broker，默认初始资金通常是 10,000 货币单位。

接下来，我们将添加数据源（DataFeed）、策略（Strategy）、指标（Indicator）等，逐步完善我们的交易系统。

