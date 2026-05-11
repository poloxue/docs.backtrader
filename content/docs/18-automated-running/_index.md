---
title: "自动运行"
weight: 18
---

# 自动运行

迄今为止，所有 backtrader 示例都是从零创建 Python 模块，加载数据、策略、观察器，并设置现金和佣金方案。

算法交易的目标是自动化。backtrader 作为回测和算法交易平台，其自动化是必然的需求。

安装 backtrader 后，提供了两个入口点用于自动化大多数任务：

1. `bt-run-py`：一个脚本。
2. `btrun`（可执行文件）：由 setuptools 在打包时创建，在 Windows 下不会出现”找不到路径/文件”的错误。

以下描述适用于这两个工具。

`btrun` 允许你：

- 指定要加载的数据源
- 设置数据格式
- 指定日期范围
- 传递参数给 Cerebro
- 禁用标准观察器
  - 这是一个遗留开关（在 Cerebro 参数实现之前）。如果通过 Cerebro 传递了标准观察器相关参数，此开关将忽略（对应 Cerebro 的 `stdstats` 参数）
- 从内置或 Python 模块加载观察器（如 DrawDown）
- 设置经纪商现金和佣金方案（佣金、保证金、倍数）
- 启用绘图，控制图表的数量和样式
- 添加参数化的 writer
- 核心功能：加载策略（内置或来自 Python 模块）
  - 传递参数给加载的策略

请参阅下面的脚本用法。

## 应用用户定义策略

考虑以下策略，它：

- 简单加载一个简单移动平均线（默认周期为 15）
- 打印输出
- 存在于名为 `mymod.py` 的文件中

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)

import backtrader as bt
import backtrader.indicators as btind

class MyTest(bt.Strategy):
    params = (('period', 15),)

    def log(self, txt, dt=None):
        '''策略的日志记录函数'''
        dt = dt or self.data.datetime[0]
        if isinstance(dt, float):
            dt = bt.num2date(dt)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        sma = btind.SMA(period=self.p.period)

    def next(self):
        ltxt = '%d, %.2f, %.2f, %.2f, %.2f, %.2f, %.2f'

        self.log(ltxt %
                 (len(self),
                  self.data.open[0], self.data.high[0],
                  self.data.low[0], self.data.close[0],
                  self.data.volume[0], self.data.openinterest[0]))
```

使用常规测试示例执行策略非常简单：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2006-day-001.txt \
      --strategy mymod.py
```

图表输出：

![Chart Output](image_path)

控制台输出：

```
2006-01-20T23:59:59+00:00, 15, 3593.16, 3612.37, 3550.80, 3550.80, 0.00, 0.00
2006-01-23T23:59:59+00:00, 16, 3550.24, 3550.24, 3515.07, 3544.31, 0.00, 0.00
2006-01-24T23:59:59+00:00, 17, 3544.78, 3553.16, 3526.37, 3532.68, 0.00, 0.00
2006-01-25T23:59:59+00:00, 18, 3532.72, 3578.00, 3532.72, 3578.00, 0.00, 0.00
...
2006-12-22T23:59:59+00:00, 252, 4109.86, 4109.86, 4072.62, 4073.50, 0.00, 0.00
2006-12-27T23:59:59+00:00, 253, 4079.70, 4134.86, 4079.70, 4134.86, 0.00, 0.00
2006-12-28T23:59:59+00:00, 254, 4137.44, 4142.06, 4125.14, 4130.66, 0.00, 0.00
2006-12-29T23:59:59+00:00, 255, 4130.12, 4142.01, 4119.94, 4119.94, 0.00, 0.00
```

相同策略，但将参数 `period` 设置为 50 的命令行：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2006-day-001.txt \
      --plot \
      --strategy mymod.py:period=50
```

图表输出：

![Chart Output](image_path)

注意：如果没有提供 `.py` 扩展名，`btrun` 会自动添加。

## 使用内置策略

backtrader 逐步内置了一些示例策略。与 `btrun` 一起提供了一个标准的简单移动平均交叉策略，名称为 `SMA_CrossOver`。

### 参数

- `fast`（默认 10）：快速移动平均线的周期
- `slow`（默认 30）：慢速移动平均线的周期

该策略在快速移动平均线上穿慢速移动平均线时买入，并在快速移动平均线下穿慢速移动平均线时卖出（仅在之前已买入的情况下）。

代码如下：

```python
from __future__ import (absolute_import, division, print_function, unicode_literals)

import backtrader as bt
import backtrader.indicators as btind

class SMA_CrossOver(bt.Strategy):
    params = (('fast', 10), ('slow', 30))

    def __init__(self):
        sma_fast = btind.SMA(period=self.p.fast)
        sma_slow = btind.SMA(period=self.p.slow)
        self.buysig = btind.CrossOver(sma_fast, sma_slow)

    def next(self):
        if self.position.size:
            if self.buysig < 0:
                self.sell()
        elif self.buysig > 0:
            self.buy()
```

标准执行：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2006-day-001.txt \
      --plot \
      --strategy :SMA_CrossOver
```

注意 `:`。标准的加载策略表示法为：`module:strategy:kwargs`。规则如下：

- 如果指定了 `module` 和 `strategy`，则使用该策略。
- 如果指定了 `module` 但未指定 `strategy`，则使用模块中找到的第一个策略。
- 如果未指定 `module`，则“strategy”被视为 backtrader 包中的策略。
- 如果指定了 `module` 和/或 `strategy`，如果存在 `kwargs`，它们将传递给相应的策略。

注意：相同的表示法和规则适用于 `--observer`、`--analyzer` 和 `--indicator` 选项，显然适用于相应的对象类型。

输出结果：

![Chart Output](image_path)

最后一个示例，添加佣金方案、现金并更改参数：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2006-day-001.txt \
      --plot \
      --cash 20000 \
      --commission 2.0 \
      --mult 10 \
      --margin 2000 \
      --strategy :SMA_CrossOver:fast=5,slow=20
```

输出结果：

![Chart Output](image_path)

我们已经回测了策略：

- 更改移动平均线周期
- 设置新的起始现金
- 为类似期货的工具设置佣金方案

观察每个条的现金持续变化，因为现金根据类似期货工具的每日变化进行调整。

## 使用无策略

这其实有些夸张——仍然会应用一个策略，但你可以省略策略类型，系统会自动添加默认的 `backtrader.Strategy`。

分析器、观察器和指标将自动注入策略中。

例如：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2006-day-001.txt \
      --cash 20000 \
      --commission 2.0 \
      --mult 10 \
      --margin 2000 \
      --nostdstats \
      --observer :Broker
```

这不会做很多事情，但可以实现以下目的：

- 在后台添加一个默认的 `backtrader.Strategy`
- Cerebro 不会实例化常规的 `stdstats` 观察器（Broker、BuySell、Trades）
- 手动添加一个 Broker 观察器

如上所述，`nostdstats` 是一个遗留参数。更新版本的 `btrun` 可以将参数直接传递给 Cerebro。等效调用如下：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2006-day-001.txt \
      --cash 20000 \
      --commission 2.0 \
      --mult 10 \
      --margin 2000 \
      --cerebro stdstats=False \
      --observer :Broker
```

## 添加分析器

`btrun` 还支持使用与选择内部/外部分析器相同的语法添加分析器。

例如，对 2005-2006 年进行夏普比率分析：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2005-2006-day-001.txt \
      --strategy :SMA_CrossOver \
      --analyzer :SharpeRatio
```

控制台输出为空。

如需打印分析器结果，需指定：

- `--pranalyzer`：默认打印（除非分析器覆盖了相应方法）
- `--ppranalyzer`：使用 `pprint` 模块美化输出

注意：这两个打印选项在 writer 成为 backtrader 一部分之前就有。使用不带 CSV 输出的 writer 可实现相同效果（且输出更优）。

扩展上述示例：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2005-2006-day-001.txt \
      --strategy :SMA_CrossOver \
      --analyzer :SharpeRatio \
      --plot \
      --pranalyzer

====================
== Analyzers
====================
##########
sharperatio
##########
{'sharperatio': 11.647332609673256}
```

好策略！（实际上纯属运气，示例中没有考虑佣金）

图表（仅显示分析器未在图表中绘制，因为分析器无法绘制，它们不是线条对象）：

![Chart Output](image_path)

使用 `writer` 参数的相同示例：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2005-2006-day-001.txt \
      --strategy :SMA_CrossOver \
      --analyzer :SharpeRatio \
      --plot \
      --writer

===============================================================================
Cerebro:
  -----------------------------------------------------------------------------
  - Datas:
    +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    - Data0:
      - Name: 2005-2006-day-001
      - Timeframe: Days
      - Compression: 1
  -----------------------------------------------------------------------------
  - Strategies:
    +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    - SMA_CrossOver:
      *************************************************************************
      - Params:
        - fast: 10
        - slow: 30
        - _movav: SMA
      *************************************************************************
      - Indicators:
        .......................................................................
        - SMA:
          - Lines: sma
          ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
          - Params:
            - period: 30
        .......................................................................
        - CrossOver:
          - Lines: crossover
          - Params: None
      *************************************************************************
      - Observers:
        .......................................................................
        - Broker:
          - Lines: cash, value
          - Params: None
        .......................................................................
        - BuySell:
          - Lines: buy, sell
          - Params: None
        .......................................................................
        - Trades:
          - Lines: pnlplus, pnlminus
          - Params: None
      *************************************************************************
      - Analyzers:
        .......................................................................
        - Value:
          - Begin: 10000.0
          - End: 10496.68
        .......................................................................
        - SharpeRatio:
          - Params: None
          ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
          - Analysis:
            - sharperatio: 11.6473326097
```

## 添加指标和观察器

与策略和分析器相同，`btrun` 还可以添加：

- 指标
- 观察器

语法与上面添加 Broker 观察器时所见完全相同。

让我们重复示例，添加一个随机指标、Broker 并查看图表（我们将更改一些参数）：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2006-day-001.txt \
      --nostdstats \
      --observer :Broker \
      --indicator :Stochastic:period_dslow=5 \
      --plot
```

图表：

![Chart Output](image_path)

## 绘图控制

上面大多数示例都使用了以下选项：

- `--plot`：激活创建默认图表
更多控制可以通过向 `--plot` 选项添加 kwargs 实现。

例如，使用蜡烛图代替默认的 LineOnClose 样式：
`--plot style="candle"`
调用如下：

```bash
btrun --csvformat btcsv \
      --data ../../datas/2006-day-001.txt \
      --nostdstats \
      --observer :Broker \
      --indicator :Stochastic:period_dslow=5 \
      --plot style=\"candle\"
```

注意：示例在 bash shell 中运行，传递参数给脚本之前会去除引号，因此需要使用反斜杠转义 `\"` 确保 `candle` 作为字符串传递。

图表：

![Chart Output](image_path)

## 脚本用法

直接从脚本：

```bash
$ btrun --help
usage: btrun-script.py [-h] --data DATA [--cerebro [kwargs]] [--nostdstats]
                       [--format {yahoocsv_unreversed,vchart,vchartcsv,yahoo,mt4csv,ibdata,sierracsv,yahoocsv,btcsv,vcdata}]
                       [--fromdate FROMDATE] [--todate TODATE]
                       [--timeframe {microseconds,seconds,weeks,months,minutes,days,years}]
                       [--compression COMPRESSION]
                       [--resample RESAMPLE | --replay REPLAY]
                       [--strategy module:name:kwargs]
                       [--signal module:signaltype:name:kwargs]
                       [--observer module:name:kwargs]
                       [--analyzer module:name:kwargs]
                       [--pranalyzer | --ppranalyzer]
                       [--indicator module:name:kwargs] [--writer [kwargs]]
                       [--cash CASH] [--commission COMMISSION]
                       [--margin MARGIN] [--mult MULT] [--interest INTEREST]
                       [--interest_long] [--slip_perc SLIP_PERC]
                       [--slip_fixed SLIP_FIXED] [--slip_open]
                       [--no-slip_match] [--slip_out] [--flush]
                       [--plot [kwargs]]

Backtrader Run Script

optional arguments:
  -h, --help            show this help message and exit
  --resample RESAMPLE, -rs RESAMPLE
                        resample with timeframe:compression values
  --replay REPLAY, -rp REPLAY
                        replay with timeframe:compression values
  --pranalyzer, -pralyzer
                        Automatically print analyzers
  --ppranalyzer, -ppralyzer
                        Automatically PRETTY print analyzers
  --plot [kwargs], -p [kwargs]
                        Plot the read data applying any kwargs passed

                        For example:

                          --plot style="candle" (to plot candlesticks)

Data options:
  --data DATA, -d DATA  Data files to be added to the system

Cerebro options:
  --cerebro [kwargs], -cer [kwargs]
                        The argument can be specified with the following form:

                          - kwargs

                            Example: "preload=True" which set its to True

                        The passed kwargs will be passed directly to the cerebro
                        instance created for the execution

                        The available kwargs to cerebro are:
                          - preload (default: True)
                          - runonce (default: True)
                          - maxcpus (default: None)
                          - stdstats (default: True)
                          - live (default: False)
                          - exactbars (default: False)
                          - preload (default: True)
                          - writer (default False)
                          - oldbuysell (default False)
                          - tradehistory (default False)
  --nostdstats          Disable the standard statistics observers
  --format {yahoocsv_unreversed,vchart,vchartcsv,yahoo,mt4csv,ibdata,sierracsv,yahoocsv,btcsv,vcdata}, --csvformat {yahoocsv_unreversed,vchart,vchartcsv,yahoo,mt4csv,ibdata,sierracsv,yahoocsv,btcsv,vcdata}, -c {yahoocsv_unreversed,vchart,vchartcsv,yahoo,mt4csv,ibdata,sierracsv,yahoocsv,btcsv,vcdata}
                        CSV Format
  --fromdate FROMDATE, -f FROMDATE
                        Starting date in YYYY-MM-DD[THH:MM:SS] format
 

 --todate TODATE, -t TODATE
                        Ending date in YYYY-MM-DD[THH:MM:SS] format
  --timeframe {microseconds,seconds,weeks,months,minutes,days,years}, -tf {microseconds,seconds,weeks,months,minutes,days,years}
                        Ending date in YYYY-MM-DD[THH:MM:SS] format
  --compression COMPRESSION, -cp COMPRESSION
                        Ending date in YYYY-MM-DD[THH:MM:SS] format

Strategy options:
  --strategy module:name:kwargs, -st module:name:kwargs
                        This option can be specified multiple times.

                        The argument can be specified with the following form:

                          - module:classname:kwargs

                            Example: mymod:myclass:a=1,b=2

                        kwargs is optional

                        If module is omitted then class name will be sought in
                        the built-in strategies module. Such as in:

                          - :name:kwargs or :name

                        If name is omitted, then the 1st strategy found in the mod
                        will be used. Such as in:

                          - module or module::kwargs

Signals:
  --signal module:signaltype:name:kwargs, -sig module:signaltype:name:kwargs
                        This option can be specified multiple times.

                        The argument can be specified with the following form:

                          - signaltype:module:signaltype:classname:kwargs

                            Example: longshort+mymod:myclass:a=1,b=2

                        signaltype may be ommited: longshort will be used

                            Example: mymod:myclass:a=1,b=2

                        kwargs is optional

                        signaltype will be uppercased to match the defintions
                        fromt the backtrader.signal module

                        If module is omitted then class name will be sought in
                        the built-in signals module. Such as in:

                          - LONGSHORT::name:kwargs or :name

                        If name is omitted, then the 1st signal found in the mod
                        will be used. Such as in:

                          - module or module:::kwargs

Observers and statistics:
  --observer module:name:kwargs, -ob module:name:kwargs
                        This option can be specified multiple times.

                        The argument can be specified with the following form:

                          - module:classname:kwargs

                            Example: mymod:myclass:a=1,b=2

                        kwargs is optional

                        If module is omitted then class name will be sought in
                        the built-in observers module. Such as in:

                          - :name:kwargs or :name

                        If name is omitted, then the 1st observer found in the
                        will be used. Such as in:

                          - module or module::kwargs

Analyzers:
  --analyzer module:name:kwargs, -an module:name:kwargs
                        This option can be specified multiple times.

                        The argument can be specified with the following form:

                          - module:classname:kwargs

                            Example: mymod:myclass:a=1,b=2

                        kwargs is optional

                        If module is omitted then class name will be sought in
                        the built-in analyzers module. Such as in:

                          - :name:kwargs or :name

                        If name is omitted, then the 1st analyzer found in the
                        will be used. Such as in:

                          - module or module::kwargs

Indicators:
  --indicator module:name:kwargs, -ind module:name:kwargs
                        This option can be specified multiple times.

                        The argument can be specified with the following form:

                          - module:classname:kwargs

                            Example: mymod:myclass:a=1,b=2

                        kwargs is optional

                        If module is omitted then class name will be sought in
                        the built-in analyzers module. Such as in:

                          - :name:kwargs or :name

                        If name is omitted, then the 1st analyzer found in the
                        will be used. Such as in:

                          - module or module::kwargs

Writers:
  --writer [kwargs], -wr [kwargs]
                        This option can be specified multiple times.

                        The argument can be specified with the following form:

                          - kwargs

                            Example: a=1,b=2

                        kwargs is optional

                        It creates a system wide writer which outputs run data

                        Please see the documentation for the available kwargs

Cash and Commission Scheme Args:
  --cash CASH, -cash CASH
                        Cash to set to the broker
  --commission COMMISSION, -comm COMMISSION
                        Commission value to set
  --margin MARGIN, -marg MARGIN
                        Margin type to set
  --mult MULT, -mul MULT
                        Multiplier to use
  --interest INTEREST   Credit Interest rate to apply (0.0x)
  --interest_long       Apply credit interest to long positions
  --slip_perc SLIP_PERC
                        Enable slippage with a percentage value
  --slip_fixed SLIP_FIXED
                        Enable slippage with a fixed point value
  --slip_open           enable slippage for when matching opening prices
  --no-slip_match       Disable slip_match, ie: matching capped at
                        high-low if slippage goes over those limits
  --slip_out            with slip_match enabled, match outside high-low
  --flush               flush the output - useful under win32 systems
```
