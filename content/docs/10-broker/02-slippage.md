---
title: "滑点"
weight: 2
---

# 滑点

回测无法完全模拟真实市场条件。无论模拟多好，真实市场中滑点仍可能发生，即请求的价格可能无法匹配。

集成的回测 Broker 支持滑点，可通过以下参数配置：

参数名        | 默认值             | 描述
------------- | ------------------ | -----------------
slip_perc     | 0.0                | 应用于买卖订单的价格上下滑动的绝对百分比（且为正值），注意：0.01 是 1%，0.001 是 0.1%；
slip_fixed    | 0.0                | 应用于买卖订单的价格上下滑动的单位百分比（且为正值），注意：如果 `slip_perc` 非零，则优先于此。
slip_open     | False              | 是否为专门使用下一个柱的开盘价执行的订单滑动价格。例如，市场订单将在下一个可用tick执行，即柱的开盘价。这也适用于其他一些执行，因为逻辑尝试检测开盘价是否会匹配请求的价格/执行类型在移动到新柱时。
slip_match    | True               | - 如果为 True，经纪商将通过在高/低价位封顶滑点来提供匹配，以防它们超出。<br/>- 如果为 False，经纪商将不会使用当前价格匹配订单，并将在下一次迭代中尝试执行
slip_limit    | True               | - 限价订单，给定确切的匹配价格请求，即使 `slip_match` 为 False，也会被匹配。<br/>- 此选项控制该行为。<br/>- 如果为 True，那么限价订单将通过在限价/高低价位封顶价格进行匹配。<br/>- 如果为 False 且滑点超出上限，则不会有匹配
slip_out      | False              | 即使价格超出高-低范围，也提供滑点。

## 工作原理

滑点应用逻辑取决于订单执行类型：

**`Close`** - 不应用滑点
- 订单匹配收盘价（当天最后一个价格），滑点无法发生，因为这是会话的最后一个价格，没有浮动空间。

**`Market`** - 应用滑点
- 注意 `slip_open` 的例外情况。市场订单匹配下一根柱的开盘价。

**`Limit`** - 按以下逻辑应用滑点
- 如果匹配价格是开盘价，根据 `slip_open` 参数决定是否应用滑点（如果应用，价格不会比限价更差）。
- 如果匹配价格不是限价，滑点在高/低点封顶，`slip_limit` 决定超过封顶时是否匹配。
- 如果匹配价格就是限价，不应用滑点。

**`Stop`** - 触发后，应用与 Market 相同的逻辑

**`StopLimit`** - 触发后，应用与 Limit 相同的逻辑

这种方法在回测和可用数据的限制范围内，提供最符合实际的模拟。

## 配置滑点

每次运行时，Cerebro 引擎已使用默认参数实例化一个 Broker。有两种方式配置滑点。

配置滑点为基于百分比的方法

```python
BackBroker.set_slippage_perc(
  perc, 
  slip_open=True,
  slip_limit=True,
  slip_match=True,
  slip_out=False,
)
```

配置滑点为固定点数的方法：

```python
BackBroker.set_slippage_fixed(
  fixed,
  slip_open=True,
  slip_limit=True,
  slip_match=True,
  slip_out=False,
)
```


替换经纪商，例如：

```python
import backtrader as bt

cerebro = bt.Cerebro()
cerebro.broker = bt.brokers.BackBroker(slip_perc=0.005)  # 0.5%
```

## 实际示例

源码中包含一个使用 Market 订单和信号的多空示例，可帮助理解逻辑。

先运行无滑点的版本作为参考基准：

```bash
$ ./slippage.py --plot
01 2005-03-22 23:59:59 SELL Size: -1 / Price: 3040.55
02 2005-04-11 23:59:59 BUY  Size: +1 / Price: 3088.47
03 2005-04-11 23:59:59 BUY  Size: +1 / Price: 3088.47
04 2005-04-19 23:59:59 SELL Size: -1 / Price: 2948.38
05 2005-04-19 23:59:59 SELL Size: -1 / Price: 2948.38
06 2005-05-19 23:59:59 BUY  Size: +1 / Price: 3034.88
...
35 2006-12-19 23:59:59 BUY  Size: +1 / Price: 4121.01
```

![image](https://example.com/image1.png)

使用配置为 1.5% 滑点的相同运行：

```bash
$ ./slippage.py --slip_perc 0.015
01 2005-03-22 23:59:59 SELL Size: -1 / Price: 3040.55
02 2005-04-11 23:59:59 BUY  Size: +1 / Price: 3088.47
03 2005-04-11 23:59:59 BUY  Size: +1 / Price: 3088.47
04 2005-04-19 23:59:59 SELL Size: -1 / Price: 2948.38
05 2005-04-19 23:59:59 SELL Size: -1 / Price: 2948.38
06 2005-05-19 23:59:59 BUY  Size: +1 / Price: 3034.88
...
35 2006-12-19 23:59:59 BUY  Size: +1 / Price: 4121.01
```

没有变化。这是预期行为。

因为执行类型为 Market，且未设置 `slip_open=True`。

市场订单匹配下一个柱的开盘价，但未允许开盘价滑动。

设置 `slip_open=True` 后运行：

```bash
$ ./slippage.py --slip_perc 0.015 --slip_open
01 2005-03-22 23:59:59 SELL Size: -1 / Price: 3021.66
02 2005-04-11 23:59:59 BUY  Size: +1 / Price: 3088.47
03 2005-04-11 23:59:59 BUY  Size: +1 / Price: 3088.47
04 2005-04-19 23:59:59 SELL Size: -1 / Price: 2948.38
05 2005-04-19 23:59:59 SELL Size: -1 / Price: 2948.38
06 2005-05-19 23:59:59 BUY  Size: +1 / Price: 3055.14
...
35 2006-12-19 23:59:59 BUY  Size: +1 / Price: 4121.01
```

可以看到价格已经发生变化。分配的价格更差或相同（如操作 35）。2016-12-19 的开盘价和最高价相同，价格不能高于最高价，否则会返回不存在的数据。

当然，如果需要，Backtrader 也允许在高-低范围外匹配。启用 `slip_out` 后运行：

```bash
$ ./slippage.py --slip_perc 0.015 --slip_open --slip_out
01 2005-03-22 23:59:59 SELL Size: -1 / Price: 2994.94
02 2005-04-11 23:59:59 BUY  Size: +1 / Price: 3134.80
03 2005-04-11 23:59:59 BUY  Size: +1 / Price: 3134.80
04 2005-04-19 23:59:59 SELL Size: -1 / Price: 2904.15
05 2005-04-19 23:59:59 SELL Size: -1 / Price: 2904.15
06 2005-05-19 23:59:59 BUY  Size: +1 / Price: 3080.40
...
35 2006-12-19 23:59:59 BUY  Size: +1 / Price: 4182.83
```

匹配价格令人震惊，明显超出范围。以操作 35 为例，匹配价格 4182.83，而图表显示资产价格从未接近该值。

`slip_match` 默认值为 True，Backtrader 会在价格封顶或未封顶时都提供匹配。禁用它：

```bash
$ ./slippage.py --slip_perc 0.015 --slip_open --no-slip_match
01 2005-04-15 23:59:59 SELL Size: -1 / Price: 3028.10
02 2005-05-18 23:59:59 BUY  Size: +1 / Price: 3029.40
03 2005-06-01 23:59:59 BUY  Size: +1 / Price: 3124.03
04 2005-10-06 23:59:59 SELL Size: -1 / Price: 3365.57
05 2005-10-06 23:59:59 SELL Size: -1 / Price: 3365.57
06 2005-12-01 23:59:59 BUY  Size: +1 / Price: 3499.95
07 2005-12-01 23:59:59 BUY  Size: +1 / Price: 3499.95
08 2006-02-28 23:59:59 SELL Size: -1 / Price: 3782.71
09 2006-02-28 23:59:59 SELL Size: -1 / Price: 3782.71
10 2006-05-23 23:59:59 BUY  Size: +1 / Price: 3594.68
11 2006-05-23 23:59:59 BUY  Size: +1 / Price: 3594.68
12 2006-11-27 23:59:59 SELL Size: -1 / Price: 3984.37
13 2006-11-27 23:59:59 SELL Size: -1 / Price: 3984.37
```

操作从 35 次降至 13 次。原理：

禁用 `slip_match` 后，当滑点将价格推高至最高价以上或最低价以下时，不允许匹配。1.5% 的滑点导致约 22 个操作无法执行。

这些例子展示了不同滑点选项如何协同工作。
