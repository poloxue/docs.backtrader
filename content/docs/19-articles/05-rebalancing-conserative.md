---
title: "保守型公式的再平衡"
weight: 5
---

# 保守型公式的再平衡

本文提出保守型公式的方法：**Python 中的保守型公式：简化的量化投资**

这只是众多再平衡方法中的一种，相对易于理解。方法概要：

- 从 Y 只股票中选出 x 只（如从 1000 只中选 100 只）
- 选股标准：
  - 低波动性
  - 高净派息收益率（Net Payout Yield，NPY）
  - 高动量
  - 每月再平衡一次

接下来展示如何在 Backtrader 中实现该策略。

### 数据

即使有获胜的策略，没有数据一切也无从谈起。因此，需要考虑数据格式和加载方式。

假设有一组 CSV 文件，每个文件包含：

- 每月 OHLCV 数据
- 额外的净派息收益率（NPY）列，形成 ohlcvn 数据集

CSV 数据格式如下：

```
date, open, high, low, close, volume, npy
2001-12-31, 1.0, 1.0, 1.0, 1.0, 0.5, 3.0
2002-01-31, 2.0, 2.5, 1.1, 1.2, 3.0, 5.0
...
```

每行表示一个月的数据。通过 Backtrader 的 CSV 数据加载引擎创建扩展类：

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

这样就完成了数据源的扩展。通过 `lines=('npy',)`，将净派息收益率数据添加到了 OHLCV 数据流中。其他常见字段（如 open、high 等）已经是 `GenericCSVData` 的一部分。通过在 `params` 中指定位置，告知 Backtrader 净派息收益率所在的列。

### 策略

将逻辑封装到 Backtrader 的标准策略中。为使其通用可自定义，采用与数据源相同的 `params` 方法。

回顾快速总结中的要点：

- 从一个包含 Y 只股票的股票池中选出 x 只

策略不负责将股票添加到池中，但负责选择。假设池中有 1000 只股票，设置了 x=100，但实际只加入了 50 只，策略仍会尝试选 100 只。为应对这种情况：

- 设置 `selperc` 参数，默认 0.10（10%），即从池中选择该比例的股票。

例如，池中有 1000 只股票则选 100 只；只有 50 只则选 5 只。

排名公式如下：

```
(momentum * net payout) / volatility
```

即动量更大、派息收益率更高、波动性更低的股票评分更高。

动量使用”变动率”指标（ROC，Rate of Change），衡量价格在一段时间内的变化比率。

净派息收益率已作为数据的一部分包含在内。

波动性使用基于 n 周期回报率的标准差计算。

有了这些信息，策略可以初始化参数并设置每月迭代中使用的指标和计算方法。

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

简单的评估方法是计算最终资产值减去初始现金。

Backtrader 还提供了内置的性能分析器，如 Sharpe 比率、加权回报率、SQN 等，可参考文档进一步分析。

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
