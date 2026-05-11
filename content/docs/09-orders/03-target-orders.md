---
title: "目标订单"
weight: 3
---

# 目标订单

{{< youtube 7jhzx0NWKrk >}}

1.8.10.96 版本之前，`buy` 和 `sell` 通过 `sizer` 确定持仓大小。但 `sizer` 无法决定操作方向（买入还是卖出），因此需要一个新的概念来加入智能决策层。这就是策略中的 `order_target_xxx` 方法家族。

受 **zipline** 启发，这些方法允许直接指定最终目标，目标类型包括：

- `size` -> 特定资产的股份或合约数量
- `value` -> 资产在投资组合中的货币价值
- `percent` -> 资产占当前投资组合的百分比

**注意**：这些方法的详细说明可在 Strategy 类的参考文档中找到。它们使用与 `buy` 和 `sell` 相同的参数签名，只是将 `size` 替换为目标参数。

这些方法的核心是：指定最终目标，由方法自动决定买入或卖出。三种方法的逻辑相同。以下是 `order_target_size` 的工作方式：

如果目标大于当前仓位，则发出买入指令，买入的数量为target - position_size。例如：

- 仓位：0，目标：7 -> buy(size=7 - 0) -> buy(size=7)
- 仓位：3，目标：7 -> buy(size=7 - 3) -> buy(size=4)
- 仓位：-3，目标：7 -> buy(size=7 - -3) -> buy(size=10)
- 仓位：-3，目标：-2 -> buy(size=-2 - -3) -> buy(size=1)

如果目标小于当前仓位，则发出卖出指令，卖出的数量为position_size - target。例如：

- 仓位：0，目标：-7 -> sell(size=0 - -7) -> sell(size=7)
- 仓位：3，目标：-7 -> sell(size=3 - -7) -> sell(size=10)
- 仓位：-3，目标：-7 -> sell(size=-3 - -7) -> sell(size=4)
- 仓位：3，目标：2 -> sell(size=3 - 2) -> sell(size=1)

当使用order_target_value设定目标值时，会考虑资产在投资组合中的当前价值和持仓量，以决定最终的操作。逻辑如下：

- 如果目标值大于当前值且仓位>=0 -> 买入
- 如果目标值大于当前值且仓位<0 -> 卖出
- 如果目标值小于当前值且仓位>=0 -> 卖出
- 如果目标值小于当前值且仓位<0 -> 买入

`order_target_percent` 的逻辑与 `order_target_value` 相同。该方法简单地根据当前投资组合的总价值确定资产的目标价值。

## 示例

**Backtrader** 为每个新功能提供示例，本例也不例外。示例在 `order_target` 子目录中，逻辑简单，用于验证结果是否符合预期：

- 奇数月（1月、3月等），使用日期作为目标（`order_target_value` 将日期乘以1000）；
- 偶数月（2月、4月等），使用31减去日期作为目标；

### order_target_size

我们来看1月和2月的情况。

```bash
$ ./order_target.py --target-size --plot
0001 - 2005-01-03 - Position Size:     00 - Value 1000000.00
0001 - 2005-01-03 - Order Target Size: 03
0002 - 2005-01-04 - Position Size:     03 - Value 999994.39
0002 - 2005-01-04 - Order Target Size: 04
0003 - 2005-01-05 - Position Size:     04 - Value 999992.48
0003 - 2005-01-05 - Order Target Size: 05
0004 - 2005-01-06 - Position Size:     05 - Value 999988.79
...
0020 - 2005-01-31 - Position Size:     28 - Value 999968.70
0020 - 2005-01-31 - Order Target Size: 31
0021 - 2005-02-01 - Position Size:     31 - Value 999954.68
0021 - 2005-02-01 - Order Target Size: 30
0022 - 2005-02-02 - Position Size:     30 - Value 999979.65
0022 - 2005-02-02 - Order Target Size: 29
0023 - 2005-02-03 - Position Size:     29 - Value 999966.33
0023 - 2005-02-03 - Order Target Size: 28
...
```

1月时，目标从年初第一个交易日的3开始递增，持仓从0增加到3，之后每次增加1。

1月结束时最后一个目标是31。进入2月后，持仓被报告，新目标变为30，然后每次递减1。

### order_target_value

期望有类似的行为：

```bash
$ ./order_target.py --target-value --plot
0001 - 2005-01-03 - Position Size:     00 - Value 1000000.00
0001 - 2005-01-03 - data value 0.00
0001 - 2005-01-03 - Order Target Value: 3000.00
0002 - 2005-01-04 - Position Size:     78 - Value 999854.14
0002 - 2005-01-04 - data value 2853.24
0002 - 2005-01-04 - Order Target Value: 4000.00
0003 - 2005-01-05 - Position Size:     109 - Value 999801.68
0003 - 2005-01-05 - data value 3938.17
0003 - 2005-01-05 - Order Target Value: 5000.00
0004 - 2005-01-06 - Position Size:     138 - Value 999699.57
...
0020 - 2005-01-31 - Position Size:     808 - Value 999206.37
0020 - 2005-01-31 - data value 28449.68
0020 - 2005-01-31 - Order Target Value: 31000.00
0021 - 2005-02-01 - Position Size:     880 - Value 998807.33
0021 - 2005-02-01 - data value 30580.00
0021 - 2005-02-01 - Order Target Value: 30000.00
0022 - 2005-02-02 - Position Size:     864 - Value 999510.21
0022 - 2005-02-02 - data value 30706.56
0022 - 2005-02-02 - Order Target Value: 29000.00
0023 - 2005-02-03 - Position Size:     816 - Value 999130.05
0023 - 2005-02-03 - data value 28633.44
0023 - 2005-02-03 - Order Target Value: 28000.00
...
```

### order_target_percent

此方法仅计算当前投资组合价值的百分比：

```bash
$ ./order_target.py --target-percent --plot
0001 - 2005-01-03 - Position Size:     00 - Value 1000000.00
0001 - 2005-01-03 - data percent 0.00
0001 - 2005-01-03 - Order Target Percent: 0.03
0002 - 2005-01-04 - Position Size:     785 - Value 998532.05
0002 - 2005-01-04 - data percent 0.03
0002 - 2005-01-04 - Order Target Percent: 0.04
0003 - 2005-01-05 - Position Size:     1091 - Value 998007.44
0003 - 2005-01-05 - data percent 0.04
0003 - 2005-01-05 - Order Target Percent: 0.05
0004 - 2005-01-06 - Position Size:     1381 - Value 996985.64
...
0020 - 2005-01-31 - Position Size:     7985 - Value 991966.28
0020 - 2005-01-31 - data percent 0.28
0020 - 2005-01-31 - Order Target Percent: 0.31
0021 - 2005-02-01 - Position Size:     8733 - Value 988008.94
0021 - 2005-02-01 - data percent 0.31
0021 - 2005-02-01 - Order Target Percent: 0.30
0022 - 2005-02-02 - Position Size:     8530 - Value 995005.45
0022 - 2005-02-02 - data percent 0.30
0022 - 2005-02-02 - Order Target Percent: 0.29
0023 - 2005-02-03 - Position Size:     8120 - Value 991240.75
0023 - 2005-02-03 - data percent 0.29
0023 - 2005-02-03 - Order Target Percent: 0.28
...
```

## 示例使用

```bash
$ ./order_target.py --help
usage: order_target.py [-h] [--data DATA] [--fromdate FROMDATE]
                       [--todate TODATE] [--cash CASH]
                       (--target-size | --target-value | --target-percent)
                       [--plot [kwargs]]

Sample for Order Target

optional arguments:
  -h, --help            show this help message and exit
  --data DATA           Specific data to be read in (default:
                        ../../datas/yhoo-1996-2015.txt)
  --fromdate FROMDATE   Starting date in YYYY-MM-DD format (default:
                        2005-01-01)
  --todate TODATE       Ending date in YYYY-MM-DD format (default: 2006-12-31)
  --cash CASH           Ending date in YYYY-MM-DD format (default: 1000000)
  --target-size         Use order_target_size (default: False)
  --target-value        Use order_target_value (default: False)
  --target-percent      Use order_target_percent (default: False)
  --plot [kwargs], -p [kwargs]
                        Plot the read data applying any kwargs passed For
                        example: --plot style="candle" (to plot candles)
                        (default: None)
```
