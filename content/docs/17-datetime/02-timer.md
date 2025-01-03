---
title: "定时器"
weight: 2
---

# 定时器

在版本 1.9.44.116 中，backtrader 添加了定时器功能。这使得在特定时间点调用 `notify_timer`（在 Cerebro 和 Strategy 中可用）成为可能，用户可以进行精细控制。

## 选项

- 基于绝对时间输入或相对于会话开始/结束时间的定时器。
- 时间规范的时区指定，可以直接通过 pytz 兼容对象或通过数据源会话结束时间进行指定。
- 相对于指定时间的起始偏移。
- 重复间隔。
- 工作日过滤器（带有结转选项）。
- 月份天数过滤器（带有结转选项）。
- 自定义回调过滤器。

## 使用模式

在 Cerebro 和 Strategy 子类中，定时器回调将在以下方法中接收：

```python
def notify_timer(self, timer, when, *args, **kwargs):
    '''接收定时器通知，其中 `timer` 是通过 `add_timer` 返回的定时器，`when` 是调用时间。
    `args` 和 `kwargs` 是传递给 `add_timer` 的额外参数。

    实际的 `when` 时间可能稍后，但系统可能无法在之前调用定时器。
    此值是定时器值，而不是系统时间。
    '''
```

## 添加定时器 - 通过 Strategy

使用以下方法完成：

```python
def add_timer(self, when,
              offset=datetime.timedelta(), repeat=datetime.timedelta(),
              weekdays=[], weekcarry=False,
              monthdays=[], monthcarry=True,
              allow=None,
              tzdata=None, cheat=False,
              *args, **kwargs):
    '''
```

它返回创建的 Timer 实例。下面是参数的解释。

## 添加定时器 - 通过 Cerebro

使用相同的方法，只是增加了参数 `strats`。如果设置为 `True`，定时器不仅会通知给 cerebro，还会通知系统中运行的所有策略。

```python
def add_timer(self, when,
              offset=datetime.timedelta(), repeat=datetime.timedelta(),
              weekdays=[], weekcarry=False,
              monthdays=[], monthcarry=True,
              allow=None,
              tzdata=None, cheat=False, strats=False,
              *args, **kwargs):
    '''
```

它返回创建的 Timer 实例。

## 定时器调用时间

### 如果 `cheat=False`

这是默认设置。在这种情况下，定时器将在以下情况后调用：

- 数据源加载当前柱的新值之后。
- 经纪人评估订单并重新计算投资组合价值之后。
- 指标重新计算之前（因为这由策略触发）。
- 调用任何策略的 `next` 方法之前。

### 如果 `cheat=True`

在这种情况下，定时器将在以下情况后调用：

- 数据源加载当前柱的新值之后。
- 经纪人评估订单并重新计算投资组合价值之前。

因此，在指标重新计算和调用任何策略的 `next` 方法之前调用定时器，这样可以实现以下场景：

- 在新柱被经纪人评估之前调用定时器。
- 指标具有前一天收盘时的值，并可用于生成进出信号（或在上次评估 `next` 时设置标志）。
- 因为新价格可用，可以使用开盘价计算股份。这假设一个人通过观察开盘拍卖得到良好的开盘指示。

## 使用每日柱

示例 `scheduled.py` 默认使用 backtrader 发行版中提供的标准每日柱。策略参数如下：

```python
class St(bt.Strategy):
    params = dict(
        when=bt.timer.SESSION_START,
        timer=True,
        cheat=False,
        offset=datetime.timedelta(),
        repeat=datetime.timedelta(),
        weekdays=[],
    )
```

数据具有以下会话时间：

- 开始：09:00
- 结束：17:30

运行只带时间的情况：

```shell
$ ./scheduled.py --strat when='datetime.time(15,30)'
```

输出结果：

```
strategy notify_timer with tid 0, when 2005-01-03 15:30:00 cheat False
1, 2005-01-03 17:30:00, Week 1, Day 1, O 2952.29, H 2989.61, L 2946.8, C 2970.02
strategy notify_timer with tid 0, when 2005-01-04 15:30:00 cheat False
2, 2005-01-04 17:30:00, Week 1, Day 2, O 2969.78, H 2979.88, L 2961.14, C 2971.12
strategy notify_timer with tid 0, when 2005-01-05 15:30:00 cheat False
3, 2005-01-05 17:30:00, Week 1, Day 3, O 2969.0, H 2969.0, L 2942.69, C 2947.19
strategy notify_timer with tid 0, when 2005-01-06 15:30:00 cheat False
...
```

指定的定时器在 15:30 触发。没有惊喜。让我们增加 30 分钟的偏移：

```shell
$ ./scheduled.py --strat when='datetime.time(15,30)',offset='datetime.timedelta(minutes=30)'
```

输出结果：

```
strategy notify_timer with tid 0, when 2005-01-03 16:00:00 cheat False
1, 2005-01-03 17:30:00, Week 1, Day 1, O 2952.29, H 2989.61, L 2946.8, C 2970.02
strategy notify_timer with tid 0, when 2005-01-04 16:00:00 cheat False
2, 2005-01-04 17:30:00, Week 1, Day 2, O 2969.78, H 2979.88, L 2961.14, C 2971.12
strategy notify_timer with tid 0, when 2005-01-05 16:00:00 cheat False
...
```

定时器的时间从 15:30 变为 16:00。没有惊喜。让我们以会话开始为参考：

```shell
$ ./scheduled.py --strat when='bt.timer.SESSION_START',offset='datetime.timedelta(minutes=30)'
```

输出结果：

```
strategy notify_timer with tid 0, when 2005-01-03 09:30:00 cheat False
1, 2005-01-03 17:30:00, Week 1, Day 1, O 2952.29, H 2989.61, L 2946.8, C 2970.02
strategy notify_timer with tid 0, when 2005-01-04 09:30:00 cheat False
2, 2005-01-04 17:30:00, Week 1, Day 2, O 2969.78, H 2979.88, L 2961.14, C 2971.12
...
```

定时器回调时间为 09:30。会话开始时间为 09:00。这样可以简单地说希望在会话开始后 30 分钟执行一个操作。

## 运行带有 5 分钟柱的数据

示例 `scheduled-min.py` 默认运行 backtrader 发行版中的标准 5 分钟柱。策略参数如下：

```python
class St(bt.Strategy):
    params = dict(
        when=bt.timer.SESSION_START,
        timer=True,
        cheat=False,
        offset=datetime.timedelta(),
        repeat=datetime.timedelta(),
        weekdays=[],
        weekcarry=False,
        monthdays=[],
        monthcarry=True,
    )
```

数据具有相同的会话时间：

- 开始：09:00
- 结束：17:30

让我们进行一些实验。首先是一个定时器。

```shell
$ ./scheduled-min.py --strat when='datetime.time(15, 30)'
```

输出结果：

```
1, 2006-01-02 09:05:00, Week 1, Day 1, O 3578.73, H 3587.88, L 3578.73, C 3582.99
2, 2006-01-02 09:10:00, Week 1, Day 1, O 3583.01, H 3588.4, L 3583.01, C 3588.03
...
77, 2006-01-02 15:25:00, Week 1, Day 1, O 3599.07, H 3599.68, L 3598.47, C 3599.68
strategy notify_timer with tid 0, when 2006-01-02 15:30:00 cheat False
78, 2006-01-02 15:30:00, Week 1, Day 1, O 3599.64, H 3599.73, L 3599.0, C 3599.67
...
179

, 2006-01-03 15:25:00, Week 1, Day 2, O 3634.72, H 3635.0, L 3634.06, C 3634.87
strategy notify_timer with tid 0, when 2006-01-03 15:30:00 cheat False
180, 2006-01-03 15:30:00, Week 1, Day 2, O 3634.81, H 3634.89, L 3634.04, C 3634.23
...
```

定时器在 15:30 触发。日志显示了它在前两天的触发情况。

## 增加 15 分钟的重复间隔

```shell
$ ./scheduled-min.py --strat when='datetime.time(15, 30)',repeat='datetime.timedelta(minutes=15)'
```

输出结果：

```
...
74, 2006-01-02 15:10:00, Week 1, Day 1, O 3596.12, H 3596.63, L 3595.92, C 3596.63
75, 2006-01-02 15:15:00, Week 1, Day 1, O 3596.36, H 3596.65, L 3596.19, C 3596.65
76, 2006-01-02 15:20:00, Week 1, Day 1, O 3596.53, H 3599.13, L 3596.12, C 3598.9
77, 2006-01-02 15:25:00, Week 1, Day 1, O 3599.07, H 3599.68, L 3598.47, C 3599.68
strategy notify_timer with tid 0, when 2006-01-02 15:30:00 cheat False
78, 2006-01-02 15:30:00, Week 1, Day 1, O 3599.64, H 3599.73, L 3599.0, C 3599.67
79, 2006-01-02 15:35:00, Week 1, Day 1, O 3599.61, H 3600.29, L 3599.52, C 3599.92
80, 2006-01-02 15:40:00, Week 1, Day 1, O 3599.96, H 3602.06, L 3599.76, C 3602.05
strategy notify_timer with tid 0, when 2006-01-02 15:45:00 cheat False
81, 2006-01-02 15:45:00, Week 1, Day 1, O 3601.97, H 3602.07, L 3601.45, C 3601.83
82, 2006-01-02 15:50:00, Week 1, Day 1, O 3601.74, H 3602.8, L 3601.63, C 3602.8
83, 2006-01-02 15:55:00, Week 1, Day 1, O 3602.53, H 3602.74, L 3602.33, C 3602.61
strategy notify_timer with tid 0, when 2006-01-02 16:00:00 cheat False
84, 2006-01-02 16:00:00, Week 1, Day 1, O 3602.58, H 3602.75, L 3601.81, C 3602.14
85, 2006-01-02 16:05:00, Week 1, Day 1, O 3602.16, H 3602.16, L 3600.86, C 3600.96
86, 2006-01-02 16:10:00, Week 1, Day 1, O 3601.2, H 3601.49, L 3600.94, C 3601.27
...
strategy notify_timer with tid 0, when 2006-01-02 17:15:00 cheat False
99, 2006-01-02 17:15:00, Week 1, Day 1, O 3603.96, H 3603.96, L 3602.89, C 3603.79
100, 2006-01-02 17:20:00, Week 1, Day 1, O 3603.94, H 3605.95, L 3603.87, C 3603.91
101, 2006-01-02 17:25:00, Week 1, Day 1, O 3604.0, H 3604.76, L 3603.85, C 3604.64
strategy notify_timer with tid 0, when 2006-01-02 17:30:00 cheat False
102, 2006-01-02 17:30:00, Week 1, Day 1, O 3604.06, H 3604.41, L 3603.95, C 3604.33
103, 2006-01-03 09:05:00, Week 1, Day 2, O 3604.08, H 3609.6, L 3604.08, C 3609.6
104, 2006-01-03 09:10:00, Week 1, Day 2, O 3610.34, H 3617.31, L 3610.34, C 3617.31
105, 2006-01-03 09:15:00, Week 1, Day 2, O 3617.61, H 3617.87, L 3616.03, C 3617.51
106, 2006-01-03 09:20:00, Week 1, Day 2, O 3617.24, H 3618.86, L 3616.09, C 3618.42
...
179, 2006-01-03 15:25:00, Week 1, Day 2, O 3634.72, H 3635.0, L 3634.06, C 3634.87
strategy notify_timer with tid 0, when 2006-01-03 15:30:00 cheat False
180, 2006-01-03 15:30:00, Week 1, Day 2, O 3634.81, H 3634.89, L 3634.04, C 3634.23
...
```

## 使用作弊模式

```shell
$ ./scheduled-min.py --strat when='bt.timer.SESSION_START',cheat=True
```

输出结果：

```
strategy notify_timer with tid 1, when 2006-01-02 09:00:00 cheat True
-- 2006-01-02 09:05:00 Create buy order
strategy notify_timer with tid 0, when 2006-01-02 09:00:00 cheat False
1, 2006-01-02 09:05:00, Week 1, Day 1, O 3578.73, H 3587.88, L 3578.73, C 3582.99
-- 2006-01-02 09:10:00 Buy Exec @ 3583.01
2, 2006-01-02 09:10:00, Week 1, Day 1, O 3583.01, H 3588.4, L 3583.01, C 3588.03
...
```

创建订单时间为 09:05:00，执行时间为 09:10:00，因为经纪人没有在开盘时作弊模式。让我们设置它……

```shell
$ ./scheduled-min.py --strat when='bt.timer.SESSION_START',cheat=True --broker coo=True
```

输出结果：

```
strategy notify_timer with tid 1, when 2006-01-02 09:00:00 cheat True
-- 2006-01-02 09:05:00 Create buy order
strategy notify_timer with tid 0, when 2006-01-02 09:00:00 cheat False
-- 2006-01-02 09:05:00 Buy Exec @ 3578.73
1, 2006-01-02 09:05:00, Week 1, Day 1, O 3578.73, H 3587.88, L 3578.73, C 3582.99
2, 2006-01-02 09:10:00, Week 1, Day 1, O 3583.01, H 3588.4, L 3583.

01, C 3588.03
...
```

下达时间和执行时间均为 09:05:00，执行价格为 09:05:00 的开盘价。

## 其他场景

定时器允许通过传递一个整数列表来指定哪些天（iso 代码，周一为 1，周日为 7）定时器可以实际触发。例如：

```python
weekdays=[5]  # 这将请求定时器仅在周五有效
```

如果周五是非交易日，并且定时器应在下一个交易日触发，可以添加 `weekcarry=True`。

类似地，可以决定每月的某一天，例如：

```python
monthdays=[15]  # 每月的 15 日
```

如果 15 日是非交易日，并且定时器应在下一个交易日触发，可以添加 `monthcarry=True`。

目前没有实现像以下规则的功能：3 月、6 月、9 月和 12 月的第三个星期五（期货/期权到期日），但可以通过传递一个回调函数来实现规则：

```python
allow=callable  # 回调函数接收一个 datetime.date 实例，如果日期适用于定时器，则返回 True，否则返回 False
```

实现上述规则的代码：

```python
class FutOpExp(object):
    def __init__(self):
        self.fridays = 0
        self.curmonth = -1

    def __call__(self, d):
        _, _, isowkday = d.isocalendar()

        if d.month != self.curmonth:
            self.curmonth = d.month
            self.fridays = 0

        # 周一=1 ... 周日=7
        if isowkday == 5 and self.curmonth in [3, 6, 9, 12]:
            self.fridays += 1

            if self.fridays == 3:  # 第三个星期五
                return True  # 定时器允许

        return False  # 定时器不允许
```

然后将 `allow=FutOpExp()` 传递给定时器的创建。

## `add_timer` 的参数

- `when`：可以是
  - `datetime.time` 实例（见下文的 `tzdata`）
  - `bt.timer.SESSION_START` 引用会话开始时间
  - `bt.timer.SESSION_END` 引用会话结束时间
- `offset`：必须是 `datetime.timedelta` 实例，用于偏移 `when` 的值。与 `SESSION_START` 和 `SESSION_END` 结合使用时很有意义，表示定时器在会话开始后 15 分钟被调用。
- `repeat`：必须是 `datetime.timedelta` 实例，指示首次调用后，在同一会话内重复调用的间隔。
- `weekdays`：一个排序的可迭代对象，包含整数，指示定时器在星期几（iso 代码，周一为 1，周日为 7）实际有效。如果未指定，定时器将在所有天数有效。
- `weekcarry`（默认值：False）：如果为 True，并且在一周内未看到指定的日期（例如交易假日），定时器将在下一天执行（即使在新的一周内）。
- `monthdays`：一个排序的可迭代对象，包含整数，指示定时器在每月的哪些天实际有效。如果未指定，定时器将在所有天数有效。
- `monthcarry`（默认值：True）：如果未看到指定日期（周末、交易假日），定时器将在下一个可用的日子执行。
- `allow`（默认值：None）：一个回调函数，接收一个 `datetime.date` 实例，如果日期适用于定时器，则返回 True，否则返回 False。
- `tzdata`：可以是 None（默认），pytz 实例或数据源实例。
  - None：`when` 按原样解释（这相当于将其视为 UTC，即使它不是）。
  - pytz 实例：`when` 将解释为在指定时区的本地时间。
  - 数据源实例：`when` 将解释为在数据源实例的 `tz` 参数指定的本地时间。
- `strats`（默认值：False）：是否调用策略的 `notify_timer`。
- `cheat`（默认值：False）：如果为 True，定时器将在经纪人有机会评估订单之前调用。这打开了在会话开始前基于开盘价下单的机会。
- `*args`：任何额外的参数将传递给 `notify_timer`。
- `**kwargs`：任何额外的关键字参数将传递给 `notify_timer`。

## 示例用法 `scheduled.py`

```shell
$ ./scheduled.py --help
```

输出结果：

```
usage: scheduled.py [-h] [--data0 DATA0] [--fromdate FROMDATE]
                    [--todate TODATE] [--cerebro kwargs] [--broker kwargs]
                    [--sizer kwargs] [--strat kwargs] [--plot [kwargs]]

Sample Skeleton

optional arguments:
  -h, --help           show this help message and exit
  --data0 DATA0        Data to read in (default:
                       ../../datas/2005-2006-day-001.txt)
  --fromdate FROMDATE  Date[time] in YYYY-MM-DD[THH:MM:SS] format (default: )
  --todate TODATE      Date[time] in YYYY-MM-DD[THH:MM:SS] format (default: )
  --cerebro kwargs     kwargs in key=value format (default: )
  --broker kwargs      kwargs in key=value format (default: )
  --sizer kwargs       kwargs in key=value format (default: )
  --strat kwargs       kwargs in key=value format (default: )
  --plot [kwargs]      kwargs in key=value format (default: )
```

## 示例源代码 `scheduled.py`

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse
import datetime

import backtrader as bt


class St(bt.Strategy):
    params = dict(
        when=bt.timer.SESSION_START,
        timer=True,
        cheat=False,
        offset=datetime.timedelta(),
        repeat=datetime.timedelta(),
        weekdays=[],
    )

    def __init__(self):
        bt.ind.SMA()
        if self.p.timer:
            self.add_timer(
                when=self.p.when,
                offset=self.p.offset,
                repeat=self.p.repeat,
                weekdays=self.p.weekdays,
            )
        if self.p.cheat:
            self.add_timer(
                when=self.p.when,
                offset=self.p.offset,
                repeat=self.p.repeat,
                cheat=True,
            )

        self.order = None

    def prenext(self):
        self.next()

    def next(self):
        _, isowk, isowkday = self.datetime.date().isocalendar()
        txt = '{}, {}, Week {}, Day {}, O {}, H {}, L {}, C {}'.format(
            len(self), self.datetime.datetime(),
            isowk, isowkday,
            self.data.open[0], self.data.high[0],
            self.data.low[0], self.data.close[0])

        print(txt)

    def notify_timer(self, timer, when, *args, **kwargs):
        print('strategy notify_timer with tid {}, when {} cheat {}'.
              format(timer.p.tid, when, timer.p.cheat))

        if self.order is None and timer.p.cheat:
            print('-- {} Create buy order'.format(self.data.datetime.date()))
            self.order = self.buy()

    def notify_order(self, order):
        if order.status == order.Completed:
            print('-- {} Buy Exec @ {}'.format(
                self.data.datetime.date(), order.executed.price))


def runstrat(args=None):
    args = parse_args(args)

    cerebro = bt.Cerebro()

    # 数据源 kwargs
    kwargs = dict(
        timeframe=bt.TimeFrame.Days,
        compression=1,
        sessionstart=datetime.time(9, 0),
        sessionend=datetime.time(17, 30),
    )

    # 解析 from/to-date
    dtfmt, tmfmt = '%Y-%m-%d', 'T%H:%M:%S'
    for a, d in ((getattr(args, x), x) for x in ['fromdate', 'todate']):
        if a:
            strpfmt = dtfmt + tmfmt * ('T' in a)
            kwargs[d] = datetime.datetime.strptime(a, strpfmt)

    # 数据源
    data0 = bt.feeds.BacktraderCSVData(dataname=args.data0, **kwargs)
    cerebro.adddata(data0)

    # 经纪人
    cerebro.broker = bt.brokers.BackBroker(**eval('dict(' + args.broker + ')'))

    # 调整器
    cerebro.addsizer(bt.sizers.FixedSize, **eval('dict(' + args.sizer + ')'))

    # 策略
    cerebro.addstrategy(St, **eval('dict(' + args.strat + ')'))

    # 执行
    cerebro.run(**eval('dict(' + args.cerebro + ')'))

    if args.plot:  # 如果请求则绘图
        cerebro.plot(**eval('dict(' + args.plot + ')'))




def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description=(
            'Sample Skeleton'
        )
    )

    parser.add_argument('--data0', default='../../datas/2005-2006-day-001.txt',
                        required=False, help='Data to read in')

    # 日期默认值
    parser.add_argument('--fromdate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--todate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--cerebro', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--broker', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--sizer', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--strat', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--plot', required=False, default='',
                        nargs='?', const='{}',
                        metavar='kwargs', help='kwargs in key=value format')

    return parser.parse_args(pargs)


if __name__ == '__main__':
    runstrat()
```


## 示例源代码 scheduled-min.py

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse
import datetime

import backtrader as bt


class St(bt.Strategy):
    params = dict(
        when=bt.timer.SESSION_START,
        timer=True,
        cheat=False,
        offset=datetime.timedelta(),
        repeat=datetime.timedelta(),
        weekdays=[],
        weekcarry=False,
        monthdays=[],
        monthcarry=True,
    )

    def __init__(self):
        bt.ind.SMA()
        if self.p.timer:
            self.add_timer(
                when=self.p.when,
                offset=self.p.offset,
                repeat=self.p.repeat,
                weekdays=self.p.weekdays,
                weekcarry=self.p.weekcarry,
                monthdays=self.p.monthdays,
                monthcarry=self.p.monthcarry,
                # tzdata=self.data0,
            )
        if self.p.cheat:
            self.add_timer(
                when=self.p.when,
                offset=self.p.offset,
                repeat=self.p.repeat,
                weekdays=self.p.weekdays,
                weekcarry=self.p.weekcarry,
                monthdays=self.p.monthdays,
                monthcarry=self.p.monthcarry,
                # tzdata=self.data0,
                cheat=True,
            )

        self.order = None

    def prenext(self):
        self.next()

    def next(self):
        _, isowk, isowkday = self.datetime.date().isocalendar()
        txt = '{}, {}, Week {}, Day {}, O {}, H {}, L {}, C {}'.format(
            len(self), self.datetime.datetime(),
            isowk, isowkday,
            self.data.open[0], self.data.high[0],
            self.data.low[0], self.data.close[0])

        print(txt)

    def notify_timer(self, timer, when, *args, **kwargs):
        print('strategy notify_timer with tid {}, when {} cheat {}'.
              format(timer.p.tid, when, timer.p.cheat))

        if self.order is None and timer.params.cheat:
            print('-- {} Create buy order'.format(
                self.data.datetime.datetime()))
            self.order = self.buy()

    def notify_order(self, order):
        if order.status == order.Completed:
            print('-- {} Buy Exec @ {}'.format(
                self.data.datetime.datetime(), order.executed.price))


def runstrat(args=None):
    args = parse_args(args)
    cerebro = bt.Cerebro()

    # 数据源 kwargs
    kwargs = dict(
        timeframe=bt.TimeFrame.Minutes,
        compression=5,
        sessionstart=datetime.time(9, 0),
        sessionend=datetime.time(17, 30),
    )

    # 解析 from/to-date
    dtfmt, tmfmt = '%Y-%m-%d', 'T%H:%M:%S'
    for a, d in ((getattr(args, x), x) for x in ['fromdate', 'todate']):
        if a:
            strpfmt = dtfmt + tmfmt * ('T' in a)
            kwargs[d] = datetime.datetime.strptime(a, strpfmt)

    # 数据源
    data0 = bt.feeds.BacktraderCSVData(dataname=args.data0, **kwargs)
    cerebro.adddata(data0)

    # 经纪人
    cerebro.broker = bt.brokers.BackBroker(**eval('dict(' + args.broker + ')'))

    # 调整器
    cerebro.addsizer(bt.sizers.FixedSize, **eval('dict(' + args.sizer + ')'))

    # 策略
    cerebro.addstrategy(St, **eval('dict(' + args.strat + ')'))

    # 执行
    cerebro.run(**eval('dict(' + args.cerebro + ')'))

    if args.plot:  # 如果请求则绘图
        cerebro.plot(**eval('dict(' + args.plot + ')'))


def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description=(
            'Timer Test Intraday'
        )
    )

    parser.add_argument('--data0', default='../../datas/2006-min-005.txt',
                        required=False, help='Data to read in')

    # 日期默认值
    parser.add_argument('--fromdate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--todate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--cerebro', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--broker', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--sizer', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--strat', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--plot', required=False, default='',
                        nargs='?', const='{}',
                        metavar='kwargs', help='kwargs in key=value format')

    return parser.parse_args(pargs)


if __name__ == '__main__':
    runstrat()
```


