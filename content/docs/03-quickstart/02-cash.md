---
title: "账户配置"
weight: 2
---

# 账户配置

{{< youtube 5_69kMaLltk >}}

本指南将介绍 Backtrader 量化回测框架中的基本配置方法，包括设置初始账户资金、配置交易佣金等核心参数。这些基础配置是进行准确回测的前提，能够帮助你模拟真实的交易环境。

## 设置初始账户资金

上节中，账户资金使用的是默认值 10,000 货币单位。这个默认值可以通过 `cerebro.broker.setcash` 方法更改。

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
初始资金: 100000.00
最终资金: 100000.00
```

## 设置交易费率（佣金）

在回测中，交易成本是影响最终收益的重要因素。Backtrader 允许你灵活配置各种佣金方案。

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

在实际回测中，佣金会直接影响最终的账户价值。

## 设置滑点规则

Backtrader 也提供了简单的滑点模拟功能。

最常用的是按比例设置，使用 `set_slippage_perc` 方法。设置 0.1% 的滑点：

```python
cerebro.broker.set_slippage_perc(0.001)
```

计算规则如下：

- 买入订单：实际成交价比预期价格高，公式：预期价格 ×（1 + 滑点比例）
- 卖出订单：实际成交价比预期价格低，公式：预期价格 ×（1 - 滑点比例）

例如市价买入，预期价格 1000，滑点 0.1%，则实际买入价为 1001。

Backtrader 也支持固定滑点，通过 `cerebro.broker.set_slippage_fixed` 方法设置。

如设置固定滑点为 1：

```python
cerebro.broker.set_slippage_fixed(1)
```

市价买入时，价格 1000 则实际买入价为 1001；价格 2000 则实际买入价为 2001。


## 小结

本文介绍了账户资金、佣金和滑点的配置方法。下一节正式开始介绍数据配置。
