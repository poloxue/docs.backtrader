---
title: "账户配置"
weight: 2
---

# 账户配置

{{< youtube 5_69kMaLltk >}}

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
初始资金: 100000.00
最终资金: 100000.00
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

在实际回测中，佣金会直接影响最终的账户价值。

## 设置滑点规则

Backtrader 也提供了简单的滑点模拟功能。

最常用的是按比例设置，使用名为 set slippage percent 的方法。

如设置一个千分之一，也就是 0.1% 的滑点：

```python
cerebro.broker.set_slippage_perc(0.001)
```

它的计算规则是这样的：

- 买入订单时，实际成交价会比你预期的价格高一点，公式：预期价格 乘以 1 + 滑点比例。
- 卖出订单时，实际成交价会比你预期的价格低一点，公式：预期价格 乘以 1 - 滑点比例。

如当执行是市价买入时，价格是 1000，如果滑点是 0.1%，则买入价就是 1001。

Backtrader 也支持设置固定值的滑点，通过 `cerebro.broker.set_slippage_fixed` 方法。

如设置固定滑点为 1。

```python
cerebro.broker.set_slippage_fixed(1)
```

如当执行是市价买入时，价格是 1000，如果滑点是 1，则买入价就是 1001。当执行是市价买入时，价格是 2000，如果滑点是 1，则买入价就是 2001。


## 最后

本文介绍了 Backtrader 的账户资金、佣金和滑点的配置，下一节正式开始介绍数据的配置。
