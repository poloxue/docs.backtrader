---
title: "目标订单"
weight: 3
---

## 目标订单

在1.8.10.96版本之前，通过策略方法buy和sell可以实现智能持仓。关键在于添加一个Sizer（大小调整器），它负责确定持仓的大小。

然而，Sizer不能决定操作是买入还是卖出。这就需要一个新的概念，在决策中加入一个小的智能层。

这就是策略中的order_target_xxx方法家族。受zipline的启发，这些方法提供了简单指定最终目标的机会，无论目标是：

- size -> 特定资产的股份或合约数量
- value -> 资产在投资组合中的货币单位价值
- percent -> 当前投资组合中资产的百分比值

**注意**：这些方法的参考文档在Strategy中可以找到。总结来说，这些方法使用与buy和sell相同的参数签名，只是将size参数替换为target参数。

这些方法的核心在于指定最终目标，然后方法决定操作是买入还是卖出。所有三种方法的逻辑相同。以下是order_target_size的工作方式：

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

order_target_percent的逻辑与order_target_value相同。该方法简单地根据当前投资组合的总价值确定资产的目标价值。

### 示例

backtrader尝试为每个新功能提供一个示例，这也不例外。该示例在samples目录下的order_target子目录中。

示例逻辑比较简单，仅用于测试结果是否如预期：

- 在奇数月（1月、3月等），使用日期作为目标（在order_target_value的情况下，将日期乘以1000）
- 在偶数月（2月、4月等），使用31减去日期作为目标

#### order_target_size

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

在1月，目标从年初第一个交易日的3开始增加。持仓量从0增加到3，然后每次增加1。

在1月结束时，最后一个目标是31，当进入2月时，该持仓量被报告出来，新的目标变为30，并以1的递减变化。

#### order_target_value

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

#### order_target_percent

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
0022 - 2005-

02-02 - Position Size:     8530 - Value 995005.45
0022 - 2005-02-02 - data percent 0.30
0022 - 2005-02-02 - Order Target Percent: 0.29
0023 - 2005-02-03 - Position Size:     8120 - Value 991240.75
0023 - 2005-02-03 - data percent 0.29
0023 - 2005-02-03 - Order Target Percent: 0.28
...
```

#### 示例使用

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
