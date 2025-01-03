---
title: "保守型公式的再平衡"
weight: 5
---

# 保守型公式的再平衡

本文提出了保守型公式的方法：**Python中的保守型公式：简化的量化投资**

这只是众多可能的再平衡方法中的一种，但它相对易于理解。方法概要如下：

- 从Y个股票（比如1000个中的100个）中选出x只股票
- 选股标准为：
  - 低波动性
  - 高净派息收益率（Net Payout Yield，NPY）
  - 高动量
  - 每月再平衡一次

了解了这些概念后，接下来我们将展示如何在Backtrader中实现这一策略。

### 数据

即使有一个获胜的策略，如果没有可用的数据，那么一切都不会成真。因此，需要考虑数据的格式和如何加载它。

假设有一组CSV文件（“逗号分隔值”），每个文件包含以下特征：

- 每月的OHLCV数据
- 额外的列包含净派息收益率（NPY），以形成一个ohlcvn数据集。

CSV数据格式如下：

```
date, open, high, low, close, volume, npy
2001-12-31, 1.0, 1.0, 1.0, 1.0, 0.5, 3.0
2002-01-31, 2.0, 2.5, 1.1, 1.2, 3.0, 5.0
...
```

即每行表示一个月的数据。接下来，可以通过Backtrader的CSV数据加载引擎创建一个简单的扩展类。

```python
class NetPayOutData(bt.feeds.GenericCSVData):
    lines = ('npy',)  # 增加一行，用于存储净派息收益率
    params = dict(
        npy=6,  # npy字段位于第6列（基于0的索引）
        dtformat='%Y-%m-%d',  # 设置日期格式为yyyy-mm-dd
        timeframe=bt.TimeFrame.Months,  # 设置时间框架为按月
        openinterest=-1,  # -1表示没有openinterest字段
    )
```

这样就完成了对数据源的扩展。注意，通过`lines=('npy',)`，已经将净派息收益率（NPY）数据添加到了OHLCV数据流中。其他常见的字段（如open、high等）已经是`GenericCSVData`的一部分。通过在`params`中指定位置，我们能够告诉Backtrader净派息收益率所在的列。

### 策略

接下来，我们将逻辑封装到Backtrader的标准策略中。为了使其尽可能通用和可自定义，我们将采用与数据源相同的`params`方法。

首先，我们来回顾一下快速总结中的一个要点：

- 从一个Y个股票的宇宙中选择x个股票

策略本身不负责将股票添加到股票宇宙中，但它负责选择股票。假设宇宙中有1000只股票，但在代码中设置了x=100，那么即使只有50只股票被加入，策略也会选择100只。为了应对这种情况，我们会做如下处理：

- 设置`selperc`参数，默认值为0.10（即10%），表示从宇宙中选择的股票数量。

例如，如果宇宙中有1000只股票，则选择100只；如果只有50只股票，则选择5只。

股票的排名公式如下：

```
(momentum * net payout) / volatility
```

即，动量更大、派息收益率更高、波动性更低的股票会有更高的评分。

动量使用“变动率”指标（ROC，Rate of Change），它衡量的是价格在一段时间内的变化比率。

净派息收益率已经作为数据的一部分包含在内。

波动性则使用股票的标准差（标准差基于n周期的回报率）来计算。

有了这些信息后，策略可以初始化所需的参数，并设置将在每个月的迭代中使用的指标和计算方法。

### 策略实现

```python
class St(bt.Strategy):
    params = dict(
        selcperc=0.10,  # 从宇宙中选择的股票比例
        rperiod=1,  # 回报率计算周期，默认为1个周期
        vperiod=36,  # 波动性回顾期，默认为36个周期
        mperiod=12,  # 动量回顾期，默认为12个周期
        reserve=0.05  # 5%的预留资本
    )

    def log(self, arg):
        print('{} {}'.format(self.datetime.date(), arg))

    def __init__(self):
        # 计算选股数量
        self.selnum = int(len(self.datas) * self.p.selcperc)

        # 每只股票的资本分配比例
        self.perctarget = (1.0 - self.p.reserve) / self.selnum

        # 计算回报率、波动性和动量
        rs = [bt.ind.PctChange(d, period=self.p.rperiod) for d in self.datas]
        vs = [bt.ind.StdDev(ret, period=self.p.vperiod) for ret in rs]
        ms = [bt.ind.ROC(d, period=self.p.mperiod) for d in self.datas]

        # 排名公式： (动量 * 净派息收益率) / 波动性
        self.ranks = {d: d.npy * m / v for d, v, m in zip(self.datas, vs, ms)}

    def next(self):
        # 按排名排序
        ranks = sorted(
            self.ranks.items(),  # 获取(d, rank)对
            key=lambda x: x[1][0],  # 使用排名（元素1）进行排序
            reverse=True,  # 按排名从高到低排序
        )
        
        # 获取排名前selnum的股票
        rtop = dict(ranks[:self.selnum])
        # 获取排名低的股票
        rbot = dict(ranks[self.selnum:])
        
        # 获取当前持有的股票
        posdata = [d for d, pos in self.getpositions().items() if pos]

        # 卖出那些不再是前排名的股票
        for d in (d for d in posdata if d not in rtop):
            self.log('Exit {} - Rank {:.2f}'.format(d._name, rbot[d][0]))
            self.order_target_percent(d, target=0.0)

        # 重新平衡已经排名前的股票
        for d in (d for d in posdata if d in rtop):
            self.log('Rebal {} - Rank {:.2f}'.format(d._name, rtop[d][0]))
            self.order_target_percent(d, target=self.perctarget)
            del rtop[d]  # 删除已处理的股票

        # 为新进入的排名前的股票设置目标订单
        for d in rtop:
            self.log('Enter {} - Rank {:.2f}'.format(d._name, rtop[d][0]))
            self.order_target_percent(d, target=self.perctarget)
```

### 运行和评估

我们需要一些额外的代码来实现数据加载和运行策略的框架。

```python
def run(args=None):
    args = parse_args(args)

    cerebro = bt.Cerebro()

    # 加载数据文件
    for fname in glob.glob(os.path.join(args.datadir, '*')):
        data = NetPayOutData(dataname=fname, **dkwargs)
        cerebro.adddata(data)

    # 添加策略
    cerebro.addstrategy(St, **eval('dict(' + args.strat + ')'))

    # 设置初始现金
    cerebro.broker.setcash(args.cash)

    cerebro.run()  # 执行策略

    # 基本的性能评估
    pnl = cerebro.broker.get_value() - args.cash
    print('Profit ... or Loss: {:.2f}'.format(pnl))
```

### 性能评估

一种简单的评估方法是计算最终资产值减去初始现金的差额。

Backtrader还提供了内置的性能分析器，如Sharpe比率、加权回报率、SQN等，具体可参考文档进行进一步的性能分析。

### 完整脚本

```python
import argparse
import datetime
import glob
import os.path
import backtrader as bt

class NetPayOutData(bt.feeds.GenericCSVData):
    lines = ('npy',)  # add a line containing the net payout yield
    params = dict(
        npy=6,  # npy field is in the 6th column (0 based index)
        dtformat='%Y-%m-%d',  # fix date format a yyyy-mm-dd
        timeframe=bt.TimeFrame.Months,  # fixed the timeframe
        openinterest=-1,  # -1 indicates there is no openinterest field
    )


class St(bt.Strategy):
    params = dict(
        selcperc=0.10,  # percentage of stocks to select from the universe
        rperiod=1,  # period for the returns calculation, default 1 period
        vperiod=36,  # lookback period for volatility - default 36 periods
        mperiod=12,  # lookback period for momentum - default 12 periods
        reserve=0.05  # 5% reserve capital
    )

    def log(self, arg):
        print('{} {}'.format(self.datetime.date(), arg))

    def __init__(self):
        # calculate 1st the amount of stocks that will be selected
        self.selnum = int(len(self.datas) * self.p.selcperc)

        # allocation perc per stock
        # reserve kept to make sure orders are not rejected due to
        # margin. Prices are calculated when known (close), but orders can only
        # be executed next day (opening price). Price can gap upwards
        self.perctarget = (1.0 - self.p.reserve) / self.selnum

        # returns, volatilities and momentums
        rs = [bt.ind.PctChange(d, period=self.p.rperiod) for d in self.datas]
        vs = [bt.ind.StdDev(ret, period=self.p.vperiod) for ret in rs]
        ms = [bt.ind.ROC(d, period=self.p.mperiod) for d in self.datas]

        # simple rank formula: (momentum * net payout) / volatility
        # the highest ranked: low vol, large momentum, large payout
        self.ranks = {d: d.npy * m / v for d, v, m in zip(self.datas, vs, ms)}

    def next(self):
        # sort data and current rank
        ranks = sorted(
            self.ranks.items(),  # get the (d, rank), pair
            key=lambda x: x[1][0],  # use rank (elem 1) and current time "0"
            reverse=True,  # highest ranked 1st ... please
        )

        # put top ranked in dict with data as key to test for presence
        rtop = dict(ranks[:self.selnum])

        # For logging purposes of stocks leaving the portfolio
        rbot = dict(ranks[self.selnum:])

        # prepare quick lookup list of stocks currently holding a position
        posdata = [d for d, pos in self.getpositions().items() if pos]

        # remove those no longer top ranked
        # do this first to issue sell orders and free cash
        for d in (d for d in posdata if d not in rtop):
            self.log('Leave {} - Rank {:.2f}'.format(d._name, rbot[d][0]))
            self.order_target_percent(d, target=0.0)

        # rebalance those already top ranked and still there
        for d in (d for d in posdata if d in rtop):
            self.log('Rebal {} - Rank {:.2f}'.format(d._name, rtop[d][0]))
            self.order_target_percent(d, target=self.perctarget)
            del rtop[d]  # remove it, to simplify next iteration

        # issue a target order for the newly top ranked stocks
        # do this last, as this will generate buy orders consuming cash
        for d in rtop:
            self.log('Enter {} - Rank {:.2f}'.format(d._name, rtop[d][0]))
            self.order_target_percent(d, target=self.perctarget)


def run(args=None):
    args = parse_args(args)

    cerebro = bt.Cerebro()

    # Data feed kwargs
    dkwargs = dict(**eval('dict(' + args.dargs + ')'))

    # Parse from/to-date
    dtfmt, tmfmt = '%Y-%m-%d', 'T%H:%M:%S'
    if args.fromdate:
        fmt = dtfmt + tmfmt * ('T' in args.fromdate)
        dkwargs['fromdate'] = datetime.datetime.strptime(args.fromdate, fmt)

    if args.todate:
        fmt = dtfmt + tmfmt * ('T' in args.todate)
        dkwargs['todate'] = datetime.datetime.strptime(args.todate, fmt)

    # add all the data files available in the directory datadir
    for fname in glob.glob(os.path.join(args.datadir, '*')):
        data = NetPayOutData(dataname=fname, **dkwargs)
        cerebro.adddata(data)

    # add strategy
    cerebro.addstrategy(St, **eval('dict(' + args.strat + ')'))

    # set the cash
    cerebro.broker.setcash(args.cash)

    cerebro.run()  # execute it all

    # Basic performance evaluation ... final value ... minus starting cash
    pnl = cerebro.broker.get_value() - args.cash
    print('Profit ... or Loss: {:.2f}'.format(pnl))


def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description=('Rebalancing with the Conservative Formula'),
    )

    parser.add_argument('--datadir', required=True,
                        help='Directory with data files')

    parser.add_argument('--dargs', default='',
                        metavar='kwargs', help='kwargs in k1=v1,k2=v2 format')

    # Defaults for dates
    parser.add_argument('--fromdate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--todate', required=False, default='',
                        help='Date[time] in YYYY-MM-DD[THH:MM:SS] format')

    parser.add_argument('--cerebro', required=False, default='',
                        metavar='kwargs', help='kwargs in k1=v1,k2=v2 format')

    parser.add_argument('--cash', default=1000000.0, type=float,
                        metavar='kwargs', help='kwargs in k1=v1,k2=v2 format')

    parser.add_argument('--strat', required=False, default='',
                        metavar='kwargs', help='kwargs in k1=v1,k2=v2 format')

    return parser.parse_args(pargs)


if __name__ == '__main__':
    run()
```

这个完整的脚本展示了如何在Backtrader中实现基于保守型公式的股票选择和再平衡策略。
