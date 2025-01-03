---
title: "基准测试"
weight: 2
---

# 基准测试

Ticket #89 是关于添加基准测试以对比一个资产的表现。这个功能非常实用，因为有些策略即使盈利也可能低于单纯追踪资产的收益。

backtrader 包含两种不同类型的对象可以帮助进行追踪：

- 观察器
- 分析器

在分析器领域，已经有一个 `TimeReturn` 对象，用于跟踪整个投资组合价值的回报演变（包括现金）。

这显然也可以是一个观察器，所以在添加一些基准测试时，也对如何将观察器和分析器组合在一起进行了工作，这两者旨在跟踪相同的内容。

**注意**

观察器和分析器之间的主要区别在于观察器的线条特性，观察器记录每个值，这使得它们适合绘图和实时查询。当然，这会消耗内存。

另一方面，分析器通过 `get_analysis` 返回一组结果，并且实现可能直到运行结束时才会提供任何结果。

## 分析器 - 基准测试

标准的 `TimeReturn` 分析器已扩展为支持跟踪数据源。涉及的两个主要参数：

- `timeframe`（默认：无） 如果为 None，将报告整个回测期间的总回报。
- `data`（默认：无） 要跟踪的参考资产，而不是投资组合价值。

**注意**

此数据必须已通过 `adddata`、`resampledata` 或 `replaydata` 添加到 cerebro 实例中。

更多详细信息和参数请参阅：分析器参考。

因此，可以像这样按年度跟踪投资组合的回报：

```python
import backtrader as bt

cerebro = bt.Cerebro()
cerebro.addanalyzer(bt.analyzers.TimeReturn, timeframe=bt.TimeFrame.Years)

# 添加数据、策略等...

results = cerebro.run()
strat0 = results[0]

# 如果没有指定名称，则名称为类名的小写形式
tret_analyzer = strat0.analyzers.getbyname('timereturn')
print(tret_analyzer.get_analysis())
```

如果我们想跟踪一个数据的回报：

```python
import backtrader as bt

cerebro = bt.Cerebro()

data = bt.feeds.OneOfTheFeeds(dataname='abcde', ...)
cerebro.adddata(data)

cerebro.addanalyzer(bt.analyzers.TimeReturn, timeframe=bt.TimeFrame.Years, data=data)

# 添加策略等...

results = cerebro.run()
strat0 = results[0]

# 如果没有指定名称，则名称为类名的小写形式
tret_analyzer = strat0.analyzers.getbyname('timereturn')
print(tret_analyzer.get_analysis())
```

如果要同时跟踪两者，最好为分析器指定名称：

```python
import backtrader as bt

cerebro = bt.Cerebro()

data = bt.feeds.OneOfTheFeeds(dataname='abcde', ...)
cerebro.adddata(data)

cerebro.addanalyzer(bt.analyzers.TimeReturn, timeframe=bt.TimeFrame.Years, data=data, _name='datareturns')

cerebro.addanalyzer(bt.analyzers.TimeReturn, timeframe=bt.TimeFrame.Years, _name='timereturns')

# 添加策略等...

results = cerebro.run()
strat0 = results[0]

# 获取分析结果
tret_analyzer = strat0.analyzers.getbyname('timereturns')
print(tret_analyzer.get_analysis())
tdata_analyzer = strat0.analyzers.getbyname('datareturns')
print(tdata_analyzer.get_analysis())
```

## 观察器 - 基准测试

感谢能够在观察器中使用分析器的底层机制，添加了两个新观察器：

- `TimeReturn`
- `Benchmark`

两者都使用 `bt.analyzers.TimeReturn` 分析器来收集结果。

与上述代码片段相比，以下是一个完整的示例，以展示它们的功能。

### 观察 TimeReturn

执行：

```sh
$ ./observer-benchmark.py --plot --timereturn --timeframe notimeframe
```

注意执行选项：

- `--timereturn` 告诉示例仅执行此操作。
- `--timeframe notimeframe` 告诉分析器考虑整个数据集而不考虑时间框架边界。

最后绘制的值是 -0.26。

起始现金（从图表中显而易见）为 50K 货币单位，策略最终以 36,970 货币单位收尾，因此价值减少了 -26%。

### 观察基准测试

因为基准测试还会显示 `timereturn` 结果，让我们在激活基准测试的情况下运行相同的操作：

```sh
$ ./observer-benchmark.py --plot --timeframe notimeframe
```

结果显示：

策略表现优于资产：-0.26 对 -0.33

这不应该是庆祝的理由，但至少很清楚策略并不比资产差。

按年度跟踪的情况：

```sh
$ ./observer-benchmark.py --plot --timeframe years
```

注意：

策略的最后一个值略有变化，从 -0.26 变为 -0.27

而资产的最后一个值则显示为 -0.35（相对于上面的 -0.33）

原因是在从 2005 年到 2006 年的过渡中，策略和基准资产在 2005 年初几乎处于起始水平。

切换到周时间框架时，整个图像发生了变化：

```sh
$ ./observer-benchmark.py --plot --timeframe weeks
```

现在：

基准观察器显示出更加紧张的情况。事物上下波动，因为现在正在跟踪投资组合和数据的每周回报。

由于在年底的最后一周没有进行任何交易，且资产几乎没有移动，最后显示的值为 0.00（最后一周之前的最后收盘价为 25.54，而样本数据收盘于 25.55，差异首先在小数点后第四位显现）。

### 观察基准测试 - 另一个数据

示例允许对比不同的数据。默认情况下使用 `--benchdata1` 基准对比 Oracle。考虑整个数据集时使用 `--timeframe notimeframe`：

```sh
$ ./observer-benchmark.py --plot --timeframe notimeframe --benchdata1
```

结果显示：

策略的结果未随 `notimeframe` 变化，仍为 -26%（-0.26）

但当与另一个数据进行基准对比时，该数据在同一期间内有 +23%（0.23） 的增长

要么需要更改策略，要么需要交易另一个更好的资产。

## 结论

现在有两种方式，使用相同的底层代码/计算，来跟踪 `TimeReturn` 和 `Benchmark`：

- 观察器（`TimeReturn` 和 `Benchmark`）
- 分析器（`TimeReturn` 和带有 `data` 参数的 `TimeReturn`）

当然，基准测试并不保证利润，只是比较。

## 示例代码

```sh
$ ./observer-benchmark.py --help
usage: observer-benchmark.py [-h] [--data0 DATA0] [--data1 DATA1]
                             [--benchdata1] [--fromdate FROMDATE]
                             [--todate TODATE] [--printout] [--cash CASH]
                             [--period PERIOD] [--stake STAKE] [--timereturn]
                             [--timeframe {months,days,notimeframe,years,None,weeks}]
                             [--plot [kwargs]]

Benchmark/TimeReturn Observers Sample

optional arguments:
  -h, --help            show this help message and exit
  --data0 DATA0         Data0 to be read in (default:
                        ../../datas/yhoo-1996-2015.txt)
  --data1 DATA1         Data1 to be read in (default:
                        ../../datas/orcl-1995-2014.txt)
  --benchdata1          Benchmark against data1 (default: False)
  --fromdate FROMDATE   Starting date in YYYY-MM-DD format (default:
                        2005-01-01)
  --todate TODATE       Ending date in YYYY-MM-DD format (default: 2006-12-31)
  --printout            Print data lines (default: False)
  --cash CASH           Cash to start with (default: 50000)
  --period PERIOD       Period for the crossover moving average (default: 30)
  --stake STAKE         Stake to apply for the buy operations (default: 1000)
  --timereturn          Use TimeReturn observer instead of Benchmark (default:
                        None)
  --timeframe {months,days,notimeframe,years,None,weeks}
                        TimeFrame to apply to the Observer (default: None)
  --plot [kwargs], -p [kwargs]
                        Plot the read data applying any kwargs passed For
                        example: --plot style="candle" (to plot candles)
                        (default: None)
```

示例代码：

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import argparse
import datetime
import random
import backtrader as bt

class St(bt.Strategy):
    params = (
        ('period', 

10),
        ('printout', False),
        ('stake', 1000),
    )

    def __init__(self):
        sma = bt.indicators.SMA(self.data, period=self.p.period)
        self.crossover = bt.indicators.CrossOver(self.data, sma)

    def start(self):
        if self.p.printout:
            txtfields = list()
            txtfields.append('Len')
            txtfields.append('Datetime')
            txtfields.append('Open')
            txtfields.append('High')
            txtfields.append('Low')
            txtfields.append('Close')
            txtfields.append('Volume')
            txtfields.append('OpenInterest')
            print(','.join(txtfields))

    def next(self):
        if self.p.printout:
            # 仅打印第一个数据... 只是检查运行情况
            txtfields = list()
            txtfields.append('%04d' % len(self))
            txtfields.append(self.data.datetime.datetime(0).isoformat())
            txtfields.append('%.2f' % self.data0.open[0])
            txtfields.append('%.2f' % self.data0.high[0])
            txtfields.append('%.2f' % self.data0.low[0])
            txtfields.append('%.2f' % self.data0.close[0])
            txtfields.append('%.2f' % self.data0.volume[0])
            txtfields.append('%.2f' % self.data0.openinterest[0])
            print(','.join(txtfields))

        if self.position:
            if self.crossover < 0.0:
                if self.p.printout:
                    print('CLOSE {} @%{}'.format(self.p.stake, self.data.close[0]))
                self.close()

        else:
            if self.crossover > 0.0:
                self.buy(size=self.p.stake)
                if self.p.printout:
                    print('BUY   {} @%{}'.format(self.p.stake, self.data.close[0]))

TIMEFRAMES = {
    None: None,
    'days': bt.TimeFrame.Days,
    'weeks': bt.TimeFrame.Weeks,
    'months': bt.TimeFrame.Months,
    'years': bt.TimeFrame.Years,
    'notimeframe': bt.TimeFrame.NoTimeFrame,
}

def runstrat(args=None):
    args = parse_args(args)

    cerebro = bt.Cerebro()
    cerebro.broker.set_cash(args.cash)

    dkwargs = dict()
    if args.fromdate:
        fromdate = datetime.datetime.strptime(args.fromdate, '%Y-%m-%d')
        dkwargs['fromdate'] = fromdate

    if args.todate:
        todate = datetime.datetime.strptime(args.todate, '%Y-%m-%d')
        dkwargs['todate'] = todate

    data0 = bt.feeds.YahooFinanceCSVData(dataname=args.data0, **dkwargs)
    cerebro.adddata(data0, name='Data0')

    cerebro.addstrategy(St, period=args.period, stake=args.stake, printout=args.printout)

    if args.timereturn:
        cerebro.addobserver(bt.observers.TimeReturn, timeframe=TIMEFRAMES[args.timeframe])
    else:
        benchdata = data0
        if args.benchdata1:
            data1 = bt.feeds.YahooFinanceCSVData(dataname=args.data1, **dkwargs)
            cerebro.adddata(data1, name='Data1')
            benchdata = data1

        cerebro.addobserver(bt.observers.Benchmark, data=benchdata, timeframe=TIMEFRAMES[args.timeframe])

    cerebro.run()

    if args.plot:
        pkwargs = dict()
        if args.plot is not True:  # 评估为 True 但不是 True
            pkwargs = eval('dict(' + args.plot + ')')  # 传递的参数

        cerebro.plot(**pkwargs)

def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description='Benchmark/TimeReturn Observers Sample')

    parser.add_argument('--data0', required=False, default='../../datas/yhoo-1996-2015.txt', help='Data0 to be read in')

    parser.add_argument('--data1', required=False, default='../../datas/orcl-1995-2014.txt', help='Data1 to be read in')

    parser.add_argument('--benchdata1', required=False, action='store_true', help=('Benchmark against data1'))

    parser.add_argument('--fromdate', required=False, default='2005-01-01', help='Starting date in YYYY-MM-DD format')

    parser.add_argument('--todate', required=False, default='2006-12-31', help='Ending date in YYYY-MM-DD format')

    parser.add_argument('--printout', required=False, action='store_true', help=('Print data lines'))

    parser.add_argument('--cash', required=False, action='store', type=float, default=50000, help=('Cash to start with'))

    parser.add_argument('--period', required=False, action='store', type=int, default=30, help=('Period for the crossover moving average'))

    parser.add_argument('--stake', required=False, action='store', type=int, default=1000, help=('Stake to apply for the buy operations'))

    parser.add_argument('--timereturn', required=False, action='store_true', default=None, help=('Use TimeReturn observer instead of Benchmark'))

    parser.add_argument('--timeframe', required=False, action='store', default=None, choices=TIMEFRAMES.keys(), help=('TimeFrame to apply to the Observer'))

    parser.add_argument('--plot', '-p', nargs='?', required=False, metavar='kwargs', const=True, help=('Plot the read data applying any kwargs passed\nFor example:\n\n  --plot style="candle" (to plot candles)\n'))

    if pargs:
        return parser.parse_args(pargs)

    return parser.parse_args()

if __name__ == '__main__':
    runstrat()
```
