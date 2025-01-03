---
title: "绘制日期范围"
weight: 2
---

# 日期范围

在 1.9.31.x 版本中，backtrader 增加了部分绘图的功能。

可以使用策略实例中保存的完整时间戳数组的索引来指定绘图范围

也可以使用实际的 `datetime.date` 或 `datetime.datetime` 实例来限制绘图范围。

仍然通过标准的 `cerebro.plot` 进行。例如：

```python
cerebro.plot(start=datetime.date(2005, 7, 1), end=datetime.date(2006, 1, 31))
```

这对人类来说是最直接的方法。具有扩展能力的人类实际上可以尝试使用时间戳的索引，如下所示：

```python
cerebro.plot(start=75, end=185)
```

下面是一个非常标准的示例，其中包含一个简单移动平均线（在数据上绘图）、一个随机指标（独立绘图）以及随机指标线的交叉。在命令行参数中传递给 `cerebro.plot` 的参数。

使用日期方法的执行：

```shell
./partial-plot.py --plot 'start=datetime.date(2005, 7, 1),end=datetime.date(2006, 1, 31)'
```

Python 中的 eval 魔法允许直接在命令行中编写 `datetime.date` 并将其映射到实际有意义的内容。输出图表如下所示：

![部分绘图示例](image)

让我们将其与完整的绘图进行比较，以查看数据是否确实从两端跳过：

```shell
./partial-plot.py --plot
```

Python 中的 eval 魔法允许直接在命令行中编写 `datetime.date` 并将其映射到实际有意义的内容。输出图表如下所示：

![完整绘图示例](image)

## 示例用法

```shell
$ ./partial-plot.py --help
usage: partial-plot.py [-h] [--data0 DATA0] [--fromdate FROMDATE]
                       [--todate TODATE] [--cerebro kwargs] [--broker kwargs]
                       [--sizer kwargs] [--strat kwargs] [--plot [kwargs]]

Sample for partial plotting

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

## 示例代码

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)

import argparse
import datetime
import backtrader as bt

class St(bt.Strategy):
    params = ()

    def __init__(self):
        bt.ind.SMA()
        stoc = bt.ind.Stochastic()
        bt.ind.CrossOver(stoc.lines.percK, stoc.lines.percD)

    def next(self):
        pass

def runstrat(args=None):
    args = parse_args(args)
    cerebro = bt.Cerebro()

    # 数据馈送 kwargs
    kwargs = dict()

    # 解析 from/to-date
    dtfmt, tmfmt = '%Y-%m-%d', 'T%H:%M:%S'
    for a, d in ((getattr(args, x), x) for x in ['fromdate', 'todate']):
        if a:
            strpfmt = dtfmt + tmfmt * ('T' in a)
            kwargs[d] = datetime.datetime.strptime(a, strpfmt)

    # 数据馈送
    data0 = bt.feeds.BacktraderCSVData(dataname=args.data0, **kwargs)
    cerebro.adddata(data0)

    # 经纪人
    cerebro.broker = bt.brokers.BackBroker(**eval('dict(' + args.broker + ')'))

    # 大小调整器
    cerebro.addsizer(bt.sizers.FixedSize, **eval('dict(' + args.sizer + ')'))

    # 策略
    cerebro.addstrategy(St, **eval('dict(' + args.strat + ')'))

    # 执行
    cerebro.run(**eval('dict(' + args.cerebro + ')'))

    if args.plot:  # 如果请求绘图，则绘制
        cerebro.plot(**eval('dict(' + args.plot + ')'))

def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description=(
            'Sample for partial plotting'
        )
    )

    parser.add_argument('--data0', default='../../datas/2005-2006-day-001.txt',
                        required=False, help='Data to read in')

    # 日期的默认值
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
