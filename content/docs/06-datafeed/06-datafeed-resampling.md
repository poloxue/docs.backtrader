---
title: "重采样"
weight: 6
---

# 重采样

当数据只以单一时间框架可用，而分析需要在不同的时间框架上进行时，就需要进行数据重采样。

实际上，“重采样”应该称为“上采样”，因为它是从源时间框架到更大的时间框架（例如：从天到周）。

backtrader内置了通过过滤器对象进行重采样的支持。尽管有几种方法可以实现这一点，但有一个简单的接口可以实现：

代替使用`cerebro.adddata(data)`将数据放入系统中，使用

```python
cerebro.resampledata(data, **kwargs)
```

有两个主要选项可以控制：

- 更改时间框架
- 压缩条数

要实现这些功能，请在调用`resampledata`时使用以下参数：

- `timeframe`（默认值：`bt.TimeFrame.Days`）：目标时间框架，必须等于或大于源时间框架。
- `compression`（默认值：1）：将选定的值“n”压缩为1个条。

让我们来看一个从每日到每周的示例，通过手工编写的脚本：

```bash
$ ./resampling-example.py --timeframe weekly --compression 1
```

输出：

（这里应有图表）

我们可以将其与原始每日数据进行比较：

```bash
$ ./resampling-example.py --timeframe daily --compression 1
```

输出：

（这里应有图表）

实现这些功能的步骤如下：

1. 像往常一样加载数据。
2. 使用带有所需参数的`resampledata`将数据传递给`cerebro`：
   - `timeframe`
   - `compression`

示例代码（完整脚本在底部）：

```python
# 加载数据
datapath = args.dataname or '../../datas/2006-day-001.txt'
data = btfeeds.BacktraderCSVData(dataname=datapath)

# 方便的字典用于时间框架参数转换
tframes = dict(
    daily=bt.TimeFrame.Days,
    weekly=bt.TimeFrame.Weeks,
    monthly=bt.TimeFrame.Months)

# 添加重采样数据而不是原始数据
cerebro.resampledata(data,
                     timeframe=tframes[args.timeframe],
                     compression=args.compression)
```

最后一个示例中，我们首先将时间框架从每日更改为每周，然后将3条压缩为1条：

```bash
$ ./resampling-example.py --timeframe weekly --compression 3
```

输出：

（这里应有图表）

从原始的256个每日条中，我们最终得到18个3周的条。具体分析如下：

52周

52 / 3 = 17.33，因此有18个条。

当然，分时数据也可以进行重采样。

重采样过滤器支持其他参数，在大多数情况下不需要更改：

- `bar2edge`（默认值：True）：使用时间边界作为目标。例如，对于“ticks -> 5 seconds”，生成的5秒条将对齐到xx:00、xx:05、xx:10……
- `adjbartime`（默认值：True）：使用边界的时间调整重采样条的时间，而不是最后看到的时间戳。例如，对于重采样到“5 seconds”，条的时间将调整为hh:MM:05，即使最后看到的时间戳是hh:MM:04.33。

注意：只有当`bar2edge`为True时，才会调整时间。如果条没有对齐到边界，调整时间没有意义。

- `rightedge`（默认值：True）：使用时间边界的右边缘设置时间。

  - 如果为False，并且压缩到5秒，重采样条的时间将在hh:MM:00和hh:MM:04之间。
  - 如果为True，使用的时间边界为hh:MM:05。

- `boundoff`（默认值：0）：将重采样/重放边界前移一个单位。例如，从1分钟重采样到15分钟，默认行为是从00:01:00到00:15:00生成一个15分钟重放/重采样条。如果`boundoff`设置为1，则边界向前推1个单位。在这种情况下，原始单位是1分钟条。因此，重采样/重放将使用00:00:00到00:14:00的条生成15分钟条。

重采样测试脚本示例代码：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse

import backtrader as bt
import backtrader.feeds as btfeeds


def runstrat():
    args = parse_args()

    # 创建 cerebro 实体
    cerebro = bt.Cerebro(stdstats=False)

    # 添加策略
    cerebro.addstrategy(bt.Strategy)

    # 加载数据
    datapath = args.dataname or '../../datas/2006-day-001.txt'
    data = btfeeds.BacktraderCSVData(dataname=datapath)

    # 方便的字典用于时间框架参数转换
    tframes = dict(
        daily=bt.TimeFrame.Days,
        weekly=bt.TimeFrame.Weeks,
        monthly=bt.TimeFrame.Months)

    # 添加重采样数据而不是原始数据
    cerebro.resampledata(data,
                         timeframe=tframes[args.timeframe],
                         compression=args.compression)

    # 运行所有内容
    cerebro.run()

    # 绘制结果
    cerebro.plot(style='bar')


def parse_args():
    parser = argparse.ArgumentParser(
        description='Pandas test script')

    parser.add_argument('--dataname', default='', required=False,
                        help='要加载的文件数据')

    parser.add_argument('--timeframe', default='weekly', required=False,
                        choices=['daily', 'weekly', 'monthly'],
                        help='要重采样到的时间框架')

    parser.add_argument('--compression', default=1, required=False, type=int,
                        help='将n个条压缩为1个')

    return parser.parse_args()


if __name__ == '__main__':
    runstrat()
```
