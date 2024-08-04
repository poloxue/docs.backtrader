---
title: "数据回放"
weight: 7
---

### 数据回放

随着时间的推移，单纯对已经完全形成并关闭的条进行策略测试已不再足够，数据回放应运而生。假设：

- 策略在时间框架X上操作（例如：每日）
- 数据以更小的时间框架Y（例如：1分钟）可用

数据回放的作用正如其名：

- 使用1分钟数据回放每日条

这并不能完全再现市场的发展，但比单独观察每日完全形成并关闭的条要好得多：

- 如果策略在每日条形成期间实时操作，那么近似条形成过程能够模拟策略在实际条件下的表现。

要实现数据回放，只需遵循backtrader的常规使用模式：

1. 加载数据源
2. 使用`replaydata`将数据传递给`cerebro`
3. 添加策略

**注意：** 在数据回放时不支持预加载，因为每个条实际上是实时构建的。任何`Cerebro`实例中都会自动禁用预加载。

可以传递给`replaydata`的参数：

- `timeframe`（默认：`bt.TimeFrame.Days`）：目标时间框架，必须等于或大于源时间框架
- `compression`（默认：1）：将选定值“n”压缩为1条

扩展参数（若无特别需要请勿修改）：

- `bar2edge`（默认：True）：使用时间边界作为闭合条的目标。例如，使用“ticks -> 5 seconds”时，生成的5秒条将对齐到xx:00、xx:05、xx:10……
- `adjbartime`（默认：False）：使用边界的时间调整传递的重采样条的时间，而不是最后看到的时间戳。
- `rightedge`（默认：True）：使用时间边界的右边缘设置时间。

为了举例说明，标准的2006年每日数据将在每周基础上进行回放。这意味着：

- 最终会有52个条，每周一个
- `Cerebro`将调用`prenext`和`next`共计255次，这是每日条的原始数量

诀窍在于：

- 在每周条形成时，策略的长度（`len(self)`）保持不变。
- 每过一周，长度增加1。

以下是示例，但首先是测试脚本的主要部分，其中加载数据并将其传递给`cerebro`进行回放，然后运行。

```python
# 加载数据
datapath = args.dataname or '../../datas/2006-day-001.txt'
data = btfeeds.BacktraderCSVData(dataname=datapath)

# 方便的字典用于时间框架参数转换
tframes = dict(
    daily=bt.TimeFrame.Days,
    weekly=bt.TimeFrame.Weeks,
    monthly=bt.TimeFrame.Months)

# 首先添加原始数据 - 较小的时间框架
cerebro.replaydata(data,
                   timeframe=tframes[args.timeframe],
                   compression=args.compression)
```

#### 示例 - 每日回放至每周
脚本调用：

```bash
$ ./replay-example.py --timeframe weekly --compression 1
```

图表无法显示后台实际发生的情况，因此我们来看一下控制台输出：

```plaintext
prenext len 1 - counter 1
prenext len 1 - counter 2
prenext len 1 - counter 3
prenext len 1 - counter 4
prenext len 1 - counter 5
prenext len 2 - counter 6
...
prenext len 9 - counter 44
prenext len 9 - counter 45
---next len 10 - counter 46
---next len 10 - counter 47
---next len 10 - counter 48
---next len 10 - counter 49
---next len 10 - counter 50
---next len 11 - counter 51
---next len 11 - counter 52
---next len 11 - counter 53
...
---next len 51 - counter 248
---next len 51 - counter 249
---next len 51 - counter 250
---next len 51 - counter 251
---next len 51 - counter 252
---next len 52 - counter 253
---next len 52 - counter 254
---next len 52 - counter 255
```

我们看到内部的`self.counter`变量跟踪每次调用`prenext`或`next`。前者在应用的简单移动平均线（SMA）产生值之前调用。后者在SMA产生值时调用。

关键：

- 策略的长度（`len(self)`）每5条（每周5个交易日）变化一次。
- 策略实际上看到的是每周条在5次更新中的发展。

这并不能完全再现市场的实际逐秒（甚至分钟、小时）的发展，但比实际观察一个条要好。

视觉输出是每周图表，这是系统测试的最终结果。

#### 示例2 - 每日到每日带压缩
当然，“回放”也可以应用于相同时间框架，但带有压缩。

控制台：

```bash
$ ./replay-example.py --timeframe daily --compression 2
prenext len 1 - counter 1
prenext len 1 - counter 2
prenext len 2 - counter 3
prenext len 2 - counter 4
prenext len 3 - counter 5
prenext len 3 - counter 6
prenext len 4 - counter 7
...
---next len 125 - counter 250
---next len 126 - counter 251
---next len 126 - counter 252
---next len 127 - counter 253
---next len 127 - counter 254
---next len 128 - counter 255
```

这次我们得到了预期的一半条数，因为请求了2倍压缩。

图表：

（这里应有图表）

### 结论
可以重建市场发展。通常有可用的更小时间框架数据，可以用于离散地回放系统操作的时间框架。

测试脚本如下：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse

import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind


class SMAStrategy(bt.Strategy):
    params = (
        ('period', 10),
        ('onlydaily', False),
    )

    def __init__(self):
        self.sma = btind.SMA(self.data, period=self.p.period)

    def start(self):
        self.counter = 0

    def prenext(self):
        self.counter += 1
        print('prenext len %d - counter %d' % (len(self), self.counter))

    def next(self):
        self.counter += 1
        print('---next len %d - counter %d' % (len(self), self.counter))


def runstrat():
    args = parse_args()

    # 创建 cerebro 实体
    cerebro = bt.Cerebro(stdstats=False)

    cerebro.addstrategy(
        SMAStrategy,
        # 策略参数
        period=args.period,
    )

    # 加载数据
    datapath = args.dataname or '../../datas/2006-day-001.txt'
    data = btfeeds.BacktraderCSVData(dataname=datapath)

    # 方便的字典用于时间框架参数转换
    tframes = dict(
        daily=bt.TimeFrame.Days,
        weekly=bt.TimeFrame.Weeks,
        monthly=bt.TimeFrame.Months)

    # 首先添加原始数据 - 较小的时间框架
    cerebro.replaydata(data,
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

    parser.add_argument('--period', default=10, required=False, type=int,
                        help='应用于指标的周期')

    return parser.parse_args()


if __name__ == '__main__':
    runstrat()
```
