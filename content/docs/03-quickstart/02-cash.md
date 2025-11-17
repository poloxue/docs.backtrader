---
title: "账户配置"
weight: 2
---

本指南将介绍 Backtrader 量化回测框架中的基本配置方法，包括设置初始账户资金、配置交易佣金等核心参数。这些基础配置是进行准确回测的前提，能够帮助您模拟真实的交易环境。

## 设置初始账户资金

上节中，账户资金使是默认值 10,000 货币单位。当然，这个默认值是可以更改的，通过 `cerebro.broker` 的 `setcash` 方法即可。

```python
cerebro.broker.setcash(100000.0)
```

## 完整示例

```python
import backtrader as bt

def main():
    cerebro = bt.Cerebro()
    cerebro.broker.setcash(100000.0)

    print(f'初始资金: {cerebro.broker.getvalue()}')

    cerebro.run()

    print(f'最终资金: {cerebro.broker.getvalue()}')

if __name__ == '__main__':
    main()
```

输出：

```
Starting Portfolio Value: 100000.00
Final Portfolio Value: 100000.00
```

## 设置交易费率（佣金）

在回测中，交易成本是影响最终收益的重要因素。Backtrader 允许您灵活配置各种佣金方案。

### 按比例设置佣金

对于股票等按交易金额比例收取佣金的品种，可以使用 `setcommission` 方法：

```python
import backtrader as bt

def main():
    cerebro = bt.Cerebro()
    cerebro.broker.setcash(100000.0)
    
    # 设置交易佣金为千分之三（0.3%）
    cerebro.broker.setcommission(commission=0.003)
    
    print(f'初始资金: {cerebro.broker.getvalue():.2f}')
    cerebro.run()
    print(f'最终资金: {cerebro.broker.getvalue():.2f}')

if __name__ == '__main__':
    main()
```

### 佣金计算说明

- `commission=0.003` 表示每笔交易收取交易金额的 0.3% 作为佣金
- 买入和卖出时都会按比例收取
- 佣金在交易执行时自动从账户中扣除

在实际回测中，佣金会直接影响最终的账户价值。由于此示例中没有实际交易，所以资金没有变化。

让我们继续进入下一节，配置数据源（DataFeed）。
