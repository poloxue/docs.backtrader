---
title: "超大内存"
weight: 1
---

# 关于回测性能和超大内存执行

最近在 Reddit 上有两个相关的帖子，启发了本文的写作：

1. 一个声称 backtrader 无法处理 160 万根 K线的帖子：reddit/r/algotrading - A performant backtesting system?
2. 另一个要求一个能回测 8000 支股票的工具：reddit/r/algotrading - Backtesting libs that supports 1000+ stocks?

其中有一位作者询问如何使用一个可以回测“超大内存”的框架，“因为显然不能将所有这些数据加载到内存中。”

我们将会在本文中讨论这些概念，结合 backtrader 来解决。

### 2M K线

为了验证这一点，首先需要生成这么多的 K线。考虑到第一个发帖者提到 77 支股票和 160 万根 K线，这意味着每支股票大约有 20,779 根 K线，因此我们将进行以下操作来使数据更加简洁：

- 为 100 支股票生成 K线数据
- 每支股票生成 20,000 根 K线
- 即：生成 100 个文件，总共 200 万根 K线。

生成数据的脚本如下：

```python
import numpy as np
import pandas as pd

COLUMNS = ['open', 'high', 'low', 'close', 'volume', 'openinterest']
CANDLES = 20000
STOCKS = 100

dateindex = pd.date_range(start='2010-01-01', periods=CANDLES, freq='15min')

for i in range(STOCKS):
    data = np.random.randint(10, 20, size=(CANDLES, len(COLUMNS)))
    df = pd.DataFrame(data * 1.01, dateindex, columns=COLUMNS)
    df = df.rename_axis('datetime')
    df.to_csv('candles{:02d}.csv'.format(i))
```

该脚本生成了 100 个文件，从 `candles00.csv` 到 `candles99.csv`。实际的数据值不重要，重要的是保持标准的日期时间格式、OHLCV 数据（包括未平仓合约）。

### 测试系统

硬件/操作系统：使用的是一台 Windows 10 15.6 英寸的笔记本，配备 Intel i7 处理器和 32GB 内存。

Python 版本：CPython 3.6.1 和 pypy3 6.0.0。

其他：持续运行的应用程序占用约 20% 的 CPU，浏览器（Chrome 102 个进程）、Edge、Word、PowerPoint、Excel 及其他一些小程序正在运行。

### backtrader 默认配置

让我们回顾一下 backtrader 的默认运行时配置：

- 如果可能，预加载所有数据源
- 如果所有数据源都可以预加载，则以批处理模式运行（命名为 runonce）
- 先计算所有指标
- 按步骤执行策略逻辑和经纪人

### 在默认批处理模式下执行挑战

我们的测试脚本（见下文完整源代码）将打开这 100 个文件并使用 backtrader 的默认配置处理它们。

```bash
$ ./two-million-candles.py
Cerebro Start Time:          2019-10-26 08:33:15.563088
Strat Init Time:             2019-10-26 08:34:31.845349
Time Loading Data Feeds:     76.28
Number of data feeds:        100
Strat Start Time:            2019-10-26 08:34:31.864349
Pre-Next Start Time:         2019-10-26 08:34:32.670352
Time Calculating Indicators: 0.81
Next Start Time:             2019-10-26 08:34:32.671351
Strat warm-up period Time:   0.00
Time to Strat Next Logic:    77.11
End Time:                    2019-10-26 08:35:31.493349
Time in Strategy Next Logic: 58.82
Total Time in Strategy:      58.82
Total Time:                  135.93
Length of data feeds:        20000
Memory Usage: A peak of 348 Mbytes was observed
```

大部分时间实际上是在预加载数据（98.63 秒），其余时间用于策略执行（包括在每次迭代中通过经纪人进行处理，耗时 73.63 秒）。总时间为 173.26 秒。

性能为：每秒 14,713 根 K线。

结论：上述第一个 Reddit 线程中声称 backtrader 无法处理 160 万根 K线的说法是错误的。

### 使用 pypy

既然有线程声称使用 pypy 没有帮助，我们来看看使用 pypy 时的表现。

```bash
$ ./two-million-candles.py
Cerebro Start Time:          2019-10-26 08:39:42.958689
Strat Init Time:             2019-10-26 08:40:31.260691
Time Loading Data Feeds:     48.30
Number of data feeds:        100
Strat Start Time:            2019-10-26 08:40:31.338692
Pre-Next Start Time:         2019-10-26 08:40:31.612688
Time Calculating Indicators: 0.27
Next Start Time:             2019-10-26 08:40:31.612688
Strat warm-up period Time:   0.00
Time to Strat Next Logic:    48.65
End Time:                    2019-10-26 08:40:40.150689
Time in Strategy Next Logic: 8.54
Total Time in Strategy:      8.54
Total Time:                  57.19
Length of data feeds:        20000
```

总时间已经从 135.93 秒降到了 57.19 秒，性能提升了超过一倍。

性能为：每秒 34,971 根 K线。

内存使用：最大内存峰值为 269 MB。

这也相较于标准 CPython 解释器，内存使用有所改善。

### 处理 2M K线时的超大内存

所有这些都可以通过 backtrader 的几种配置选项进一步优化，其中包括优化缓冲区并只使用最小的数据集（理想情况下，缓冲区大小为 1，这通常发生在理想场景下）。

优化选项为 `exactbars=True`。关于 `exactbars` 的文档中有如下描述：

- `True` 或 `1`：所有“线”对象将减少内存使用至自动计算的最小周期。
- 如果某个简单移动平均线周期为 30，则底层数据将始终保留一个包含 30 根 K线的运行缓冲区，以便计算该简单移动平均线。

这一设置会禁用 `preload` 和 `runonce`。

为了进一步优化，并因为绘图会被禁用，以下设置也将被使用：`stdstats=False`，禁用标准观察者（用于现金、资产价值和交易，绘图不再需要）。

```bash
$ ./two-million-candles.py --cerebro exactbars=False,stdstats=False
Cerebro Start Time:          2019-10-26 08:37:08.014348
Strat Init Time:             2019-10-26 08:38:21.850392
Time Loading Data Feeds:     73.84
Number of data feeds:        100
Strat Start Time:            2019-10-26 08:38:21.851394
Pre-Next Start Time:         2019-10-26 08:38:21.857393
Time Calculating Indicators: 0.01
Next Start Time:             2019-10-26 08:38:21.857393
Strat warm-up period Time:   0.00
Time to Strat Next Logic:    73.84
End Time:                    2019-10-26 08:39:02.334936
Time in Strategy Next Logic: 40.48
Total Time in Strategy:      40.48
Total Time:                  114.32
Length of data feeds:        20000
```

性能为：每秒 17,494 根 K线。

内存使用：固定为 75 MB。

与之前未经优化的运行对比，预加载数据的时间已经不再是瓶颈，回测过程几乎可以立刻开始。

总时间为 114.32 秒，相较于 135.93 秒，改善了 15.90%。

内存使用改善了 68.5%。

### 再次使用 pypy

现在我们已经知道如何优化，接下来再次用 pypy 来做测试。

```bash
$ ./two-million-candles.py --cerebro exactbars=True,stdstats=False
Cerebro Start Time:          2019-10-26 08:44:32.309689
Strat Init Time:             2019-10

-26 08:45:22.177687
Time Loading Data Feeds:     58.80
Number of data feeds:        100
Strat Start Time:            2019-10-26 08:45:22.178689
Pre-Next Start Time:         2019-10-26 08:45:22.181688
Time Calculating Indicators: 0.26
Next Start Time:             2019-10-26 08:45:22.181688
Strat warm-up period Time:   0.00
Time to Strat Next Logic:    58.79
End Time:                    2019-10-26 08:45:42.316689
Time in Strategy Next Logic: 20.14
Total Time in Strategy:      20.14
Total Time:                  78.90
Length of data feeds:        20000
```

性能：每秒 50,956 根 K线。
