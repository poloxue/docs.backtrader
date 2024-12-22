---
title: "账户资金"
weight: 2
---

# 设置初始账户资金

上节中，账户资金使是默认值 10,000 货币单位。当然，这个默认值是可以更改的，通过 `cerebro.broker` 的 `setcash` 方法即可。

```python
cerebro.broker.setcash(100000.0)
```

## 完整示例

```python
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.broker.setcash(100000.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

输出：

```
Starting Portfolio Value: 100000.00
Final Portfolio Value: 100000.00
```

让我们继续进入下一节，配置数据源（DataFeed）。

