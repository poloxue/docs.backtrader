---
title: "TA-Lib"
weight: 4
---

# TA-Lib

即使backtrader已经提供了大量内置指标，并且开发一个指标主要是定义输入、输出并以自然方式编写公式，但有些人仍然希望使用TA-LIB。因为，指标X在TA-LIB库中存在，但在backtrader中不存在（作者很乐意接受请求），还有，TA-LIB的行为是众所周知的，人们信赖传统的事物。

为了满足每个人的需求，提供了TA-LIB集成。

## 需求

- TA-LIB的Python封装
- 任何需要的依赖项（例如numpy）
- 安装详情在GitHub仓库中

## 使用TA-LIB

与使用backtrader内置指标一样简单。以下是一个简单移动平均线的示例。首先是backtrader的示例：

```python
import backtrader as bt

class MyStrategy(bt.Strategy):
    params = (('period', 20),)

    def __init__(self):
        self.sma = bt.indicators.SMA(self.data, period=self.p.period)
        ...
```

接下来是TA-LIB的示例：

```python
import backtrader as bt

class MyStrategy(bt.Strategy):
    params = (('period', 20),)

    def __init__(self):
        self.sma = bt.talib.SMA(self.data, timeperiod=self.p.period)
        ...
```

注意，TA-LIB指标的参数由库本身定义，而不是backtrader。在这种情况下，TA-LIB中的SMA使用名为`timeperiod`的参数来定义操作窗口的大小。

对于需要多个输入的指标，例如随机指标：

```python
import backtrader as bt

class MyStrategy(bt.Strategy):
    params = (('period', 20),)

    def __init__(self):
        self.stoc = bt.talib.STOCH(self.data.high, self.data.low, self.data.close,
                                   fastk_period=14, slowk_period=3, slowd_period=3)
        ...
```

注意，`high`、`low`和`close`分别传递。可以尝试传递`open`而不是`low`（或其他任何数据系列）进行实验。

TA-LIB指标文档会自动解析并添加到backtrader文档中。你也可以查看TA-LIB的源代码/文档，或者执行以下操作：

```python
print(bt.talib.SMA.__doc__)
```

输出如下：

```
SMA([input_arrays], [timeperiod=30])

Simple Moving Average (Overlap Studies)

Inputs:
    price: (any ndarray)
Parameters:
    timeperiod: 30
Outputs:
    real
```

这提供了一些信息：

- 期望的输入（忽略`ndarray`的注释，因为backtrader在后台进行转换）
- 哪些参数及其默认值
- 指标实际提供的输出线

## 移动平均线和MA_Type
要选择特定的移动平均线，如`bt.talib.STOCH`，标准的TA-LIB `MA_Type`可以通过`backtrader.talib.MA_Type`访问。例如：

```python
import backtrader as bt
print('SMA:', bt.talib.MA_Type.SMA)
print('T3:', bt.talib.MA_Type.T3)
```

## 绘制TA-LIB指标
与常规用法一样，绘制TA-LIB指标无需特殊操作。

**注意：**

输出蜡烛图的指标（所有寻找蜡烛图模式的指标）会生成二进制输出：0或100。为了避免在图表上添加子图，存在自动绘图转换，以在识别模式时将其绘制在数据上。

## 示例和比较

以下是一些比较TA-LIB指标输出与backtrader内置指标的示例。注意：

- TA-LIB指标在图上有一个`TA_`前缀。这是样例特意这样做以帮助用户区分。
- 如果两者结果相同，移动平均线会叠加在现有移动平均线上，无法单独查看，这样的测试是通过的。

所有样例包括`CDLDOJI`指标作为参考。

### KAMA（考夫曼移动平均线）

这是第一个示例，因为这是样本直接比较的所有指标中唯一存在差异的：

- 样本的初始值不同
- 在某个时间点，值会趋同，两个KAMA实现具有相同的行为。

分析TA-LIB源码后发现：

- TA-LIB的实现为KAMA的初始值做了一个非行业标准的选择。
- 源代码引用：这里使用昨天的价格作为前一个KAMA。

backtrader采用了通常的选择，例如Stockcharts：

- StockCharts上的KAMA

由于需要一个初始值来开始计算，第一个KAMA只是一个简单移动平均线。因此存在差异。此外：

- TA-LIB的KAMA实现不允许指定用于调整Kaufman定义的可缩放常数的快慢周期。

样本执行：

```
$ ./talibtest.py --plot --ind kama
输出
```

图像

### SMA

```
$ ./talibtest.py --plot --ind sma
输出
```

图像

### EMA

```
$ ./talibtest.py --plot --ind ema
输出
```

图像

### 随机指标

```
$ ./talibtest.py --plot --ind stoc
输出
```

图像

### RSI

```
$ ./talibtest.py --plot --ind rsi
输出
```

图像

#### MACD

```
$ ./talibtest.py --plot --ind macd
输出
```

图像

### 布林带

```
$ ./talibtest.py --plot --ind bollinger
输出
```

图像

### AROON

注意，TA-LIB选择先绘制下降线，并且颜色与backtrader内置指标相反。

```
$ ./talibtest.py --plot --ind aroon
输出
```

图像

### Ultimate Oscillator

```
$ ./talibtest.py --plot --ind ultimate
输出
```

图像

### Trix

```
$ ./talibtest.py --plot --ind trix
输出
```

图像

### ADXR

backtrader同时提供ADX和ADXR线。

```
$ ./talibtest.py --plot --ind adxr
输出
```

图像

#### DEMA

```
$ ./talibtest.py --plot --ind dema
输出
```

图像

### TEMA

```
$ ./talibtest.py --plot --ind tema
输出
```

图像

### PPO

backtrader不仅提供PPO线，还提供更传统的MACD方法。

```
$ ./talibtest.py --plot --ind ppo
输出
```

图像

#### WilliamsR

```
$ ./talibtest.py --plot --ind williamsr
输出
```

图像

#### ROC
所有指标应具有相同的形状，但跟踪动量或变化率有多种定义。

```
$ ./talibtest.py --plot --ind roc
输出
```

图像

## 样本用法

```
$ ./talibtest.py --help
usage: talibtest.py [-h] [--data0 DATA0] [--fromdate FROMDATE]
                    [--todate TODATE]
                    [--ind {sma,ema,stoc,rsi,macd,bollinger,aroon,ultimate,trix,kama,adxr,dema,tema,ppo,williamsr,roc}]
                    [--no-doji] [--use-next] [--plot [kwargs]]

Sample for ta-lib

optional arguments:
  -h, --help            show this help message and exit
  --data0 DATA0         Data to be read in (default:
                        ../../datas/yhoo-1996-2015.txt)
  --fromdate FROMDATE   Starting date in YYYY-MM-DD format (default:
                        2005-01-01)
  --todate TODATE       Ending date in YYYY-MM-DD format (default: 2006-12-31)
  --ind {sma,ema,stoc,rsi,macd,bollinger,aroon,ultimate,trix,kama,adxr,dema,tema,ppo,williamsr,roc}
                        Which indicator pair to show together (default: sma)
  --no-doji             Remove Doji CandleStick pattern checker (default:
                        False)
  --use-next            Use next (step by step) instead of once (batch)
                        (default: False)
  --plot [kwargs], -p [kwargs]
                        Plot the read data applying any kwargs passed For
                        example (escape the quotes if needed): --plot
                        style="candle" (to plot candles) (default: None)
```

## 样本代码

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse
import datetime

import backtrader as bt


class TALibStrategy(bt.Strategy):
    params = (('ind', 'sma'), ('doji', True),)

    INDS = ['sma', 'ema', 'stoc', 'rsi', 'mac

d', 'bollinger', 'aroon',
            'ultimate', 'trix', 'kama', 'adxr', 'dema', 'ppo', 'tema',
            'roc', 'williamsr']

    def __init__(self):
        if self.p.doji:
            bt.talib.CDLDOJI(self.data.open, self.data.high,
                             self.data.low, self.data.close)

        if self.p.ind == 'sma':
            bt.talib.SMA(self.data.close, timeperiod=25, plotname='TA_SMA')
            bt.indicators.SMA(self.data, period=25)
        elif self.p.ind == 'ema':
            bt.talib.EMA(timeperiod=25, plotname='TA_SMA')
            bt.indicators.EMA(period=25)
        elif self.p.ind == 'stoc':
            bt.talib.STOCH(self.data.high, self.data.low, self.data.close,
                           fastk_period=14, slowk_period=3, slowd_period=3,
                           plotname='TA_STOCH')

            bt.indicators.Stochastic(self.data)

        elif self.p.ind == 'macd':
            bt.talib.MACD(self.data, plotname='TA_MACD')
            bt.indicators.MACD(self.data)
            bt.indicators.MACDHisto(self.data)
        elif self.p.ind == 'bollinger':
            bt.talib.BBANDS(self.data, timeperiod=25,
                            plotname='TA_BBANDS')
            bt.indicators.BollingerBands(self.data, period=25)

        elif self.p.ind == 'rsi':
            bt.talib.RSI(self.data, plotname='TA_RSI')
            bt.indicators.RSI(self.data)

        elif self.p.ind == 'aroon':
            bt.talib.AROON(self.data.high, self.data.low, plotname='TA_AROON')
            bt.indicators.AroonIndicator(self.data)

        elif self.p.ind == 'ultimate':
            bt.talib.ULTOSC(self.data.high, self.data.low, self.data.close,
                            plotname='TA_ULTOSC')
            bt.indicators.UltimateOscillator(self.data)

        elif self.p.ind == 'trix':
            bt.talib.TRIX(self.data, timeperiod=25,  plotname='TA_TRIX')
            bt.indicators.Trix(self.data, period=25)

        elif self.p.ind == 'adxr':
            bt.talib.ADXR(self.data.high, self.data.low, self.data.close,
                          plotname='TA_ADXR')
            bt.indicators.ADXR(self.data)

        elif self.p.ind == 'kama':
            bt.talib.KAMA(self.data, timeperiod=25, plotname='TA_KAMA')
            bt.indicators.KAMA(self.data, period=25)

        elif self.p.ind == 'dema':
            bt.talib.DEMA(self.data, timeperiod=25, plotname='TA_DEMA')
            bt.indicators.DEMA(self.data, period=25)

        elif self.p.ind == 'ppo':
            bt.talib.PPO(self.data, plotname='TA_PPO')
            bt.indicators.PPO(self.data, _movav=bt.indicators.SMA)

        elif self.p.ind == 'tema':
            bt.talib.TEMA(self.data, timeperiod=25, plotname='TA_TEMA')
            bt.indicators.TEMA(self.data, period=25)

        elif self.p.ind == 'roc':
            bt.talib.ROC(self.data, timeperiod=12, plotname='TA_ROC')
            bt.talib.ROCP(self.data, timeperiod=12, plotname='TA_ROCP')
            bt.talib.ROCR(self.data, timeperiod=12, plotname='TA_ROCR')
            bt.talib.ROCR100(self.data, timeperiod=12, plotname='TA_ROCR100')
            bt.indicators.ROC(self.data, period=12)
            bt.indicators.Momentum(self.data, period=12)
            bt.indicators.MomentumOscillator(self.data, period=12)

        elif self.p.ind == 'williamsr':
            bt.talib.WILLR(self.data.high, self.data.low, self.data.close,
                           plotname='TA_WILLR')
            bt.indicators.WilliamsR(self.data)


def runstrat(args=None):
    args = parse_args(args)

    cerebro = bt.Cerebro()

    dkwargs = dict()
    if args.fromdate:
        fromdate = datetime.datetime.strptime(args.fromdate, '%Y-%m-%d')
        dkwargs['fromdate'] = fromdate

    if args.todate:
        todate = datetime.datetime.strptime(args.todate, '%Y-%m-%d')
        dkwargs['todate'] = todate

    data0 = bt.feeds.YahooFinanceCSVData(dataname=args.data0, **dkwargs)
    cerebro.adddata(data0)

    cerebro.addstrategy(TALibStrategy, ind=args.ind, doji=not args.no_doji)

    cerebro.run(runcone=not args.use_next, stdstats=False)
    if args.plot:
        pkwargs = dict(style='candle')
        if args.plot is not True:  # evals to True but is not True
            npkwargs = eval('dict(' + args.plot + ')')  # args were passed
            pkwargs.update(npkwargs)

        cerebro.plot(**pkwargs)


def parse_args(pargs=None):

    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description='Sample for sizer')

    parser.add_argument('--data0', required=False,
                        default='../../datas/yhoo-1996-2015.txt',
                        help='Data to be read in')

    parser.add_argument('--fromdate', required=False,
                        default='2005-01-01',
                        help='Starting date in YYYY-MM-DD format')

    parser.add_argument('--todate', required=False,
                        default='2006-12-31',
                        help='Ending date in YYYY-MM-DD format')

    parser.add_argument('--ind', required=False, action='store',
                        default=TALibStrategy.INDS[0],
                        choices=TALibStrategy.INDS,
                        help=('Which indicator pair to show together'))

    parser.add_argument('--no-doji', required=False, action='store_true',
                        help=('Remove Doji CandleStick pattern checker'))

    parser.add_argument('--use-next', required=False, action='store_true',
                        help=('Use next (step by step) '
                              'instead of once (batch)'))

    # Plot options
    parser.add_argument('--plot', '-p', nargs='?', required=False,
                        metavar='kwargs', const=True,
                        help=('Plot the read data applying any kwargs passed\n'
                              '\n'
                              'For example (escape the quotes if needed):\n'
                              '\n'
                              '  --plot style="candle" (to plot candles)\n'))

    if pargs is not None:
        return parser.parse_args(pargs)

    return parser.parse_args()


if __name__ == '__main__':
    runstrat()
```

这样，您就可以在backtrader中使用TA-LIB的指标，并根据需要进行绘图和比较。
