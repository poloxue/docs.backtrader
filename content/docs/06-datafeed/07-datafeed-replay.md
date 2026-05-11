---
title: "数据回放"
weight: 7
---

# 数据回放

仅对已完成的 Bar 进行策略测试已不够，数据回放应运而生。假设策略在时间框架 X（如每日）上操作，而数据在更小的时间框架 Y（如 1 分钟）上可用。

数据回放正如其名，使用 1 分钟数据来重建每日 Bar。虽然不能完全再现市场发展，但比观察已完成的每日 Bar 要好得多。这可以近似模拟策略在每日 Bar 形成过程中的实际表现。

实现数据回放，按常规使用 backtrader 即可。

- 加载数据源
- 用 `replaydata` 将数据传递给 `cerebro`
- 添加策略

**注意：** 数据回放不支持预加载，因为每个 Bar 实际上是实时构建的，任何 `Cerebro` 实例中都会自动禁用预加载。

可传递给`replaydata`的参数：

参数          | 默认值               | 描述
------------- | -------------------- | -----------------------------
`timeframe`   | `bt.TimeFrame.Days`  | 目标时间框架，必须等于或大于源时间框架
`compression` | 1                    | 将选定值“n”压缩为1条

扩展参数（若无特别需要请勿修改）：

参数          | 默认值               | 描述
------------- | -------------------- | -----------------------------
`bar2edge`    | True                 | 使用时间边界作为闭合条的目标。例如，使用“ticks -> 5 seconds”时，生成的5秒条将对齐到xx:00、xx:05、xx:10……
`adjbartime`  | False                | 使用边界的时间调整传递的重采样条的时间，而不是最后看到的时间戳。
`rightedge`   | True                 | 使用时间边界的右边缘设置时间。

举例说明，标准的 2006 年每日数据按每周进行回放。

- 最终有 52 个 Bar，每周一个；
- `Cerebro` 会调用 `prenext` 和 `next` 共计 255 次（原始每日 Bar 的数量）；

关键点：

- 每周 Bar 形成过程中，策略长度（`len(self)`）保持不变。
- 每周结束后，长度增加 1。

以下是测试脚本的主要部分，加载数据并传递给 `cerebro` 进行回放：

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

## 示例 - 每日回放至每周

脚本调用：

```bash
$ ./replay-example.py --timeframe weekly --compression 1
```

图表无法显示后台的实际过程，下面看一下控制台输出：

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

内部的 `self.counter` 变量跟踪每次 `prenext` 或 `next` 调用。前者在 SMA 产生值之前调用，后者在 SMA 产生值之后调用。

- 策略长度（`len(self)`）每 5 条（每周 5 个交易日）变化一次。
- 策略实际看到的是每周 Bar 在 5 次更新中的形成过程。

这不能完全再现市场的逐秒发展，但比仅观察完成的 Bar 要好。

最终的视觉输出是每周图表。

## 示例2 - 每日压缩

“回放”也可以应用于相同时间框架加上压缩。

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

## 结论

可以利用更小时间框架的数据，离散地重建系统操作时间框架内的市场发展。

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
