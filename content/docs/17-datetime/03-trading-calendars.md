---
title: "交易日历"
weight: 3
---

# 交易日历

版本 1.9.42.116 增加了对交易日历的支持。这在以下场景中的重采样时非常有用：

- 从每日到每周的重采样现在可以将每周的K线与本周的最后一根K线一起交付。
- 这是因为交易日历可以识别下一个交易日，并且可以提前识别出本周的最后一个交易日。
- 当交易会话的结束时间不是常规时间（可以通过数据源来指定）时，从子日内到每日的重采样。

## 交易日历接口

有一个基类 `TradingCalendarBase` 用作任何交易日历的基类。它定义了两个必须被重写的方法：

```python
class TradingCalendarBase(with_metaclass(MetaParams, object)):
    def _nextday(self, day):
        '''
        返回在 `day`（datetime/date 实例）之后的下一个交易日（datetime/date 实例）以及 isocalendar 组件。
        返回值是一个包含两个组件的元组：(nextday, (y, w, d))，其中 (y, w, d)。
        '''
        raise NotImplementedError

    def schedule(self, day):
        '''
        返回给定日期（datetime/date 实例）的开盘和收盘时间（`datetime.time`）。
        '''
        raise NotImplementedError
```

## 实现

### PandasMarketCalendar

这个实现基于一个不错的包，这是从 Quantopian 的初始功能衍生出来的。包位于 `pandas_market_calendars`，可以很容易地安装：

```bash
pip install pandas_market_calendars
```

实现的接口如下：

```python
class PandasMarketCalendar(TradingCalendarBase):
    '''
    `pandas_market_calendars` 的交易日历包装器。必须安装 `pandas_market_calendar` 包。

    参数：

      - `calendar` (默认 `None`)

        参数 `calendar` 接受以下内容：

        - 字符串：支持的日历名称，例如 `NYSE`。包装器会尝试获取一个日历实例。
        - 日历实例：由 `get_calendar('NYSE')` 返回。

      - `cachesize` (默认 `365`)

        缓存查找提前天数。

    参见：

      - https://github.com/rsheftel/pandas_market_calendars
      - http://pandas-market-calendars.readthedocs.io/
    '''
    params = (
        ('calendar', None),  # 一个 pandas_market_calendars 实例或交易所名称
        ('cachesize', 365),  # 缓存查找提前天数
    )
```

### TradingCalendar

这个实现允许通过指定假期、早市天数、非交易工作日以及开盘和收盘时间来构建一个日历：

```python
class TradingCalendar(TradingCalendarBase):
    '''
    交易日历的包装器。必须安装 `pandas_market_calendars` 包。

    参数：

      - `open` (默认 `time.min`)

        常规开盘时间。

      - `close` (默认 `time.max`)

        常规收盘时间。

      - `holidays` (默认 `[]`)

        非交易日列表（`datetime.datetime` 实例）。

      - `earlydays` (默认 `[]`)

        确定日期和开盘/收盘时间的不符合常规交易时间的天数列表，每个元组包含 (`datetime.datetime`, `datetime.time`, `datetime.time`)。

      - `offdays` (默认 `ISOWEEKEND`)

        一周中市场不交易的工作日的 ISO 格式列表（周一：1 -> 周日：7）。这通常是周六和周日，因此为默认值。
    '''
    params = (
        ('open', time.min),
        ('close', _time_max),
        ('holidays', []),  # 非交易日列表（日期）
        ('earlydays', []),  # 元组列表（日期，开盘时间，收盘时间）
        ('offdays', ISOWEEKEND),  # 非交易日列表（ISO 工作日）
    )
```

## 使用模式

### 全局交易日历

通过 `Cerebro`，可以添加一个全局日历，作为所有数据源的默认日历，除非为数据源指定了一个日历：

```python
def addcalendar(self, cal):
    '''向系统添加全局交易日历。个别数据源可以有单独的日历覆盖全局日历。

    `cal` 可以是 `TradingCalendar` 的一个实例、一个字符串或一个 `pandas_market_calendars` 的实例。字符串将被实例化为 `PandasMarketCalendar`（需要在系统中安装 `pandas_market_calendar` 模块）。

    如果传递的是 `TradingCalendarBase` 的子类（而不是实例），则会被实例化。
    '''
```

### 每个数据源

通过指定一个 `calendar` 参数，遵循与上面描述的 `addcalendar` 相同的约定。例如：

```python
...
data = bt.feeds.YahooFinanceData(dataname='YHOO', calendar='NYSE', ...)
cerebro.adddata(data)
...
```

## 示例

### 从每日到每周

让我们看一个示例代码的运行结果。2016 年的复活节星期五（2016-03-25）也是纽约证券交易所的假期。如果运行没有交易日历的示例代码，让我们看看该日期前后的情况。

在这种情况下，重采样是从每日到每周（使用 YHOO 和 2016 年的每日数据）：

```bash
$ ./tcal.py

...
Strategy len 56 datetime 2016-03-23 Data0 len 56 datetime 2016-03-23 Data1 len 11 datetime 2016-03-18
Strategy len 57 datetime 2016-03-24 Data0 len 57 datetime 2016-03-24 Data1 len 11 datetime 2016-03-18
Strategy len 58 datetime 2016-03-28 Data0 len 58 datetime 2016-03-28 Data1 len 12 datetime 2016-03-24
...
```

在这个输出中，第一个日期是由策略计算的日期。第二个日期是每日数据的日期。

如预期的那样，周在 2016-03-24（星期四）结束，但是由于没有交易日历，重采样代码无法知道这一点，并且会在 2016-03-18（前一周）的日期交付重采样条形图。当交易移至 2016-03-28（星期一）时，重采样器检测到周变化，并在 2016-03-24 交付一个重采样条形图。

如果使用 NYSE 的 PandasMarketCalendar 并添加一个绘图，再次运行：

```bash
$ ./tcal.py --plot --pandascal NYSE

...
Strategy len 56 datetime 2016-03-23 Data0 len 56 datetime 2016-03-23 Data1 len 11 datetime 2016-03-18
Strategy len 57 datetime 2016-03-24 Data0 len 57 datetime 2016-03-24 Data1 len 12 datetime 2016-03-24
Strategy len 58 datetime 2016-03-28 Data0 len 58 datetime 2016-03-28 Data1 len 12 datetime 2016-03-24
...
```

有所变化！由于有日历，重采样器知道周在 2016-03-24 结束，并在同一天交付相应的每周重采样条形图。

绘图结果如下。

image

由于某些信息可能并不总是可用，可以手动编写日历。对于 NYSE 和 2016 年，日历定义如下：

```python
class NYSE_2016(bt.TradingCalendar):
    params = dict(
        holidays=[
            datetime.date(2016, 1, 1),
            datetime.date(2016, 1, 18),
            datetime.date(2016, 2, 15),
            datetime.date(2016, 3, 25),
            datetime.date(2016, 5, 30),
            datetime.date(2016, 7, 4),
            datetime.date(2016, 9, 5),
            datetime.date(2016, 11, 24),
            datetime.date(2016, 12, 26),
        ]
    )
```

复活节星期五（2016-03-25）被列为假期之一。现在运行示例代码：

```bash
$ ./tcal.py --plot --owncal

...
Strategy len 56 datetime 2016-03-23 Data0 len 56 datetime 2016-03-23 Data1 len 11 datetime 2016-03-18
Strategy len 57 datetime 2016-03-24 Data0 len 57 datetime 2016-03-24 Data1 len 12 datetime 2016-03-24
Strategy len 58 datetime 2016-03-28 Data0 len 58 datetime 2016-03-28 Data1 len 12 datetime 201

6-03-24
...
```

使用手动编写的日历定义得到了相同的结果。

### 从分钟到每日

使用一些私有的日内数据，并且知道市场在 2016-11-25 提前收盘（感恩节后的第二天市场在美国东部时间 13:00 收盘），再进行一个测试运行，这次使用第二个示例。

注意

源数据直接来自显示数据，并且在 CET 时区，即使标的资产 YHOO 在美国交易。代码中使用了 `tzinput='CET'` 和 `tz='US/Eastern'` 来让平台适当地转换输入并显示输出。

首先，不使用交易日历：

```bash
$ ./tcal-intra.py

...
Strategy len 6838 datetime 2016-11-25 18:00:00 Data0 len 6838 datetime 2016-11-25 13:00:00 Data1 len 21 datetime 2016-11-23 16:00:00
Strategy len 6839 datetime 2016-11-25 18:01:00 Data0 len 6839 datetime 2016-11-25 13:01:00 Data1 len 21 datetime 20 16-11-23 16:00:00
Strategy len 6840 datetime 2016-11-28 14:31:00 Data0 len 6840 datetime 2016-11-28 09:31:00 Data1 len 22 datetime 2016-11-25 16:00:00
Strategy len 6841 datetime 2016-11-28 14:32:00 Data0 len 6841 datetime 2016-11-28 09:32:00 Data1 len 22 datetime 2016-11-25 16:00:00
...
```

如预期的那样，这一天在 13:00 提前结束，但重采样器不知道这一点（官方会话在 16:00 结束），并继续交付前一天（2016-11-23）的重采样日线图，新重采样日线图首次在下一个交易日（2016-11-28）交付，日期为 2016-11-25。

注意

数据在 13:01 有一个额外的分钟条，这可能是由于拍卖过程中在市场关闭时间后提供的最后价格。

我们可以向流中添加一个过滤器，以过滤掉会话时间之外的条形图（过滤器将从交易日历中找出时间）。

但这不是此示例的重点。

使用 PandasMarketCalendar 实例再次运行：

```bash
$ ./tcal-intra.py --pandascal NYSE

...
Strategy len 6838 datetime 2016-11-25 18:00:00 Data0 len 6838 datetime 2016-11-25 13:00:00 Data1 len 15 datetime 2016-11-25 13:00:00
Strategy len 6839 datetime 2016-11-25 18:01:00 Data0 len 6839 datetime 2016-11-25 13:01:00 Data1 len 15 datetime 2016-11-25 13:00:00
Strategy len 6840 datetime 2016-11-28 14:31:00 Data0 len 6840 datetime 2016-11-28 09:31:00 Data1 len 15 datetime 2016-11-25 13:00:00
Strategy len 6841 datetime 2016-11-28 14:32:00 Data0 len 6841 datetime 2016-11-28 09:32:00 Data1 len 15 datetime 2016-11-25 13:00:00
...
```

现在，当日线条在 2016-11-25 的 13:00 交付时（忽略 13:01 的条形图），重采样代码通过交易日历知道这一天结束了。

让我们添加一个手动编写的定义。与前面的相同，但扩展了一些早市天数：

```python
class NYSE_2016(bt.TradingCalendar):
    params = dict(
        holidays=[
            datetime.date(2016, 1, 1),
            datetime.date(2016, 1, 18),
            datetime.date(2016, 2, 15),
            datetime.date(2016, 3, 25),
            datetime.date(2016, 5, 30),
            datetime.date(2016, 7, 4),
            datetime.date(2016, 9, 5),
            datetime.date(2016, 11, 24),
            datetime.date(2016, 12, 26),
        ],
        earlydays=[
            (datetime.date(2016, 11, 25),
             datetime.time(9, 30), datetime.time(13, 1))
        ],
        open=datetime.time(9, 30),
        close=datetime.time(16, 0),
    )
```

运行：

```bash
$ ./tcal-intra.py --owncal

...
Strategy len 6838 datetime 2016-11-25 18:00:00 Data0 len 6838 datetime 2016-11-25 13:00:00 Data1 len 16 datetime 2016-11-25 13:01:00
Strategy len 6839 datetime 2016-11-25 18:01:00 Data0 len 6839 datetime 2016-11-25 13:01:00 Data1 len 16 datetime 2016-11-25 13:01:00
Strategy len 6840 datetime 2016-11-28 14:31:00 Data0 len 6840 datetime 2016-11-28 09:31:00 Data1 len 16 datetime 2016-11-25 13:01:00
Strategy len 6841 datetime 2016-11-28 14:32:00 Data0 len 6841 datetime 2016-11-28 09:32:00 Data1 len 16 datetime 2016-11-25 13:01:00
...
```

细心的读者会注意到手动编写的定义将 2016-11-25 的结束时间定义为 13:01（使用 `datetime.time(13, 1)`）。这只是为了展示手动编写的 `TradingCalendar` 如何帮助适应。

现在 2016-11-25 的重采样日线条在 13:01 与 1 分钟条一起交付。

## 策略的额外奖励

第一个日期时间，属于策略的，总是在不同的时区，实际上是 UTC。使用这个版本 1.9.42.116，这也可以同步。以下参数已添加到 `Cerebro`（在实例化期间使用或与 `cerebro.run` 一起使用：

- `tz`（默认：`None`）

  添加策略的全局时区。参数 `tz` 可以是：
  
  - `None`：在这种情况下，策略显示的日期时间将是 UTC，这一直是标准行为。
  - `pytz` 实例。它将用于将 UTC 时间转换为所选时区。
  - `string`。尝试实例化一个 `pytz` 实例。
  - `integer`。使用相应数据在 `self.datas` 可迭代对象中的相同时区（`0` 将使用 `data0` 的时区）。

也可以通过 `cerebro.addtz` 方法支持：

```python
def addtz(self, tz):
    '''
    这也可以通过参数 `tz` 完成。
    
    添加策略的全局时区。参数 `tz` 可以是：
    
      - `None`：在这种情况下，策略显示的日期时间将是 UTC，这一直是标准行为。
      - `pytz` 实例。它将用于将 UTC 时间转换为所选时区。
      - `string`。尝试实例化一个 `pytz` 实例。
      - `integer`。使用相应数据在 `self.datas` 可迭代对象中的相同时区（`0` 将使用 `data0` 的时区）。
    '''
```

重复上一次的日内示例运行，并使用 0 作为 tz（与 data0 的时区同步），输出如下，关注与上面相同的日期和时间：

```bash
$ ./tcal-intra.py --owncal --cerebro tz=0

...
Strategy len 6838 datetime 2016-11-25 13:00:00 Data0 len 6838 datetime 2016-11-25 13:00:00 Data1 len 15 datetime 2016-11-23 16:00:00
Strategy len 6839 datetime 2016-11-25 13:01:00 Data0 len 6839 datetime 2016-11-25 13:01:00 Data1 len 16 datetime 2016-11-25 13:01:00
Strategy len 6840 datetime 2016-11-28 09:31:00 Data0 len 6840 datetime 2016-11-28 09:31:00 Data

1 len 16 datetime 2016-11-25 13:01:00
Strategy len 6841 datetime 2016-11-28 09:32:00 Data0 len 6841 datetime 2016-11-28 09:32:00 Data1 len 16 datetime 2016-11-25 13:01:00
...
```

时间戳现在与时区对齐。

## 示例使用（tcal.py）

```bash
$ ./tcal.py --help
usage: tcal.py [-h] [--data0 DATA0] [--offline] [--fromdate FROMDATE]
               [--todate TODATE] [--cerebro kwargs] [--broker kwargs]
               [--sizer kwargs] [--strat kwargs] [--plot [kwargs]]
               [--pandascal PANDASCAL | --owncal]
               [--timeframe {Weeks,Months,Years}]

Trading Calendar Sample

optional arguments:
  -h, --help            show this help message and exit
  --data0 DATA0         Data to read in (default: YHOO)
  --offline             Read from disk with same name as ticker (default: False)
  --fromdate FROMDATE   Date[time] in YYYY-MM-DD[THH:MM:SS] format (default: 2016-01-01)
  --todate TODATE       Date[time] in YYYY-MM-DD[THH:MM:SS] format (default: 2016-12-31)
  --cerebro kwargs      kwargs in key=value format (default: )
  --broker kwargs       kwargs in key=value format (default: )
  --sizer kwargs        kwargs in key=value format (default: )
  --strat kwargs        kwargs in key=value format (default: )
  --plot [kwargs]       kwargs in key=value format (default: )
  --pandascal PANDASCAL Name of trading calendar to use (default: )
  --owncal              Apply custom NYSE 2016 calendar (default: False)
  --timeframe {Weeks,Months,Years} Timeframe to resample to (default: Weeks)
```

## 示例使用（tcal-intra.py）

```bash
$ ./tcal-intra.py --help
usage: tcal-intra.py [-h] [--data0 DATA0] [--fromdate FROMDATE]
                     [--todate TODATE] [--cerebro kwargs] [--broker kwargs]
                     [--sizer kwargs] [--strat kwargs] [--plot [kwargs]]
                     [--pandascal PANDASCAL | --owncal] [--timeframe {Days}]

Trading Calendar Sample

optional arguments:
  -h, --help            show this help message and exit
  --data0 DATA0         Data to read in (default: yhoo-2016-11.csv)
  --fromdate FROMDATE   Date[time] in YYYY-MM-DD[THH:MM:SS] format (default: 2016-01-01)
  --todate TODATE       Date[time] in YYYY-MM-DD[THH:MM:SS] format (default: 2016-12-31)
  --cerebro kwargs      kwargs in key=value format (default: )
  --broker kwargs       kwargs in key=value format (default: )
  --sizer kwargs        kwargs in key=value format (default: )
  --strat kwargs        kwargs in key=value format (default: )
  --plot [kwargs]       kwargs in key=value format (default: )
  --pandascal PANDASCAL Name of trading calendar to use (default: )
  --owncal              Apply custom NYSE 2016 calendar (default: False)
  --timeframe {Days}    Timeframe to resample to (default: Days)
```

## 示例代码（tcal.py）

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import argparse
import datetime
import backtrader as bt

class NYSE_2016(bt.TradingCalendar):
    params = dict(
        holidays=[
            datetime.date(2016, 1, 1),
            datetime.date(2016, 1, 18),
            datetime.date(2016, 2, 15),
            datetime.date(2016, 3, 25),
            datetime.date(2016, 5, 30),
            datetime.date(2016, 7, 4),
            datetime.date(2016, 9, 5),
            datetime.date(2016, 11, 24),
            datetime.date(2016, 12, 26),
        ]
    )

class St(bt.Strategy):
    params = dict()

    def __init__(self):
        pass

    def start(self):
        self.t0 = datetime.datetime.utcnow()

    def stop(self):
        t1 = datetime.datetime.utcnow()
        print('Duration:', t1 - self.t0)

    def prenext(self):
        self.next()

    def next(self):
        print('Strategy len {} datetime {}'.format(len(self), self.datetime.date()), end=' ')
        print('Data0 len {} datetime {}'.format(len(self.data0), self.data0.datetime.date()), end=' ')
        if len(self.data1):
            print('Data1 len {} datetime {}'.format(len(self.data1), self.data1.datetime.date()))
        else:
            print()

def runstrat(args=None):
    args = parse_args(args)
    cerebro = bt.Cerebro()

    # 数据源参数
    kwargs = dict()

    # 解析 from/to 日期
    dtfmt, tmfmt = '%Y-%m-%d', 'T%H:%M:%S'
    for a, d in ((getattr(args, x), x) for x in ['fromdate', 'todate']):
        if a:
            strpfmt = dtfmt + tmfmt * ('T' in a)
            kwargs[d] = datetime.datetime.strptime(a, strpfmt)

    YahooData = bt.feeds.YahooFinanceData
    if args.offline:
        YahooData = bt.feeds.YahooFinanceCSVData  # 切换为从文件读取

    # 数据源
    data0 = YahooData(dataname=args.data0, **kwargs)
    cerebro.adddata(data0)

    d1 = cerebro.resampledata(data0, timeframe=getattr(bt.TimeFrame, args.timeframe))
    d1.plotinfo.plotmaster = data0
    d1.plotinfo.sameaxis = True

    if args.pandascal:
        cerebro.addcalendar(args.pandascal)
    elif args.owncal:
        cerebro.addcalendar(NYSE_2016)

    # 经纪商
    cerebro.broker = bt.brokers.BackBroker(**eval('dict(' + args.broker + ')'))

    # 大小调整器
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
            'Trading Calendar Sample'
        )
    )

    parser.add_argument('--data0', default='YHOO', required=False, help='Data to read in')
    parser.add_argument('--offline', required=False, action='store_true', help='Read from disk with same name as ticker')

    # 日期默认值
    parser.add_argument('--fromdate', required=False, default='2016-01-01', help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')
    parser.add_argument('--todate', required=False, default='2016-12-31', help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--cerebro', required=False, default='', metavar='kwargs', help='kwargs in key=value format')
    parser.add_argument('--broker', required=False, default='', metavar='kwargs', help='kwargs in key=value format')
    parser.add_argument('--sizer', required=False, default='', metavar='kwargs', help='kwargs in key=value format')
    parser.add_argument('--strat', required=False, default='', metavar='kwargs', help='kwargs in key=value format')
    parser.add_argument('--plot', required=False, default='', nargs='?', const='{}', metavar='kwargs', help='kwargs in key=value format')

    pgroup = parser.add_mutually_exclusive_group(required=False)
    pgroup.add_argument('--pandascal', required=False, action='store', default='', help='Name of trading calendar to use')
    pgroup.add_argument('--owncal', required=False, action='store_true', help='Apply custom NYSE 2016 calendar')

    parser.add_argument('--timeframe', required=False, action='store', default='Weeks', choices=['Weeks', 'Months', 'Years'], help='Timeframe to resample to')

    return parser.parse_args(pargs)

if __name__ == '__main__':
    runstrat()
```

## 示例代码（tcal-intra.py）

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)
import argparse
import datetime
import backtrader as bt

class NYSE_2016(bt.TradingCalendar

):
    params = dict(
        holidays=[
            datetime.date(2016, 1, 1),
            datetime.date(2016, 1, 18),
            datetime.date(2016, 2, 15),
            datetime.date(2016, 3, 25),
            datetime.date(2016, 5, 30),
            datetime.date(2016, 7, 4),
            datetime.date(2016, 9, 5),
            datetime.date(2016, 11, 24),
            datetime.date(2016, 12, 26),
        ],
        earlydays=[
            (datetime.date(2016, 11, 25), datetime.time(9, 30), datetime.time(13, 1))
        ],
        open=datetime.time(9, 30),
        close=datetime.time(16, 0),
    )

class St(bt.Strategy):
    params = dict()

    def __init__(self):
        pass

    def prenext(self):
        self.next()

    def next(self):
        print('Strategy len {} datetime {}'.format(len(self), self.datetime.datetime()), end=' ')
        print('Data0 len {} datetime {}'.format(len(self.data0), self.data0.datetime.datetime()), end=' ')
        if len(self.data1):
            print('Data1 len {} datetime {}'.format(len(self.data1), self.data1.datetime.datetime()))
        else:
            print()

def runstrat(args=None):
    args = parse_args(args)
    cerebro = bt.Cerebro()

    # 数据源参数
    # kwargs = dict(tz='US/Eastern')
    # import pytz
    # tz = tzinput = pytz.timezone('Europe/Berlin')
    tzinput = 'Europe/Berlin'
    # tz = tzinput
    tz = 'US/Eastern'
    kwargs = dict(tzinput=tzinput, tz=tz)

    # 解析 from/to 日期
    dtfmt, tmfmt = '%Y-%m-%d', 'T%H:%M:%S'
    for a, d in ((getattr(args, x), x) for x in ['fromdate', 'todate']):
        if a:
            strpfmt = dtfmt + tmfmt * ('T' in a)
            kwargs[d] = datetime.datetime.strptime(a, strpfmt)

    # 数据源
    data0 = bt.feeds.BacktraderCSVData(dataname=args.data0, **kwargs)
    cerebro.adddata(data0)

    d1 = cerebro.resampledata(data0, timeframe=getattr(bt.TimeFrame, args.timeframe))
    # d1.plotinfo.plotmaster = data0
    # d1.plotinfo.sameaxis = False

    if args.pandascal:
        cerebro.addcalendar(args.pandascal)
    elif args.owncal:
        cerebro.addcalendar(NYSE_2016())  # 或者 NYSE_2016() 传递一个实例

    # 经纪商
    cerebro.broker = bt.brokers.BackBroker(**eval('dict(' + args.broker + ')'))

    # 大小调整器
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
            'Trading Calendar Sample'
        )
    )

    parser.add_argument('--data0', default='yhoo-2016-11.csv', required=False, help='Data to read in')

    # 日期默认值
    parser.add
