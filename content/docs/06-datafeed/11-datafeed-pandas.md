---
title: "Pandas 数据源示例"
weight: 11
---

# Pandas 数据源示例

**注意**，需要安装 pandas 及其依赖项。支持 Pandas Dataframe 对很多人来说很重要，他们依赖于 Pandas 提供的不同数据源（包括 CSV）的解析代码及其他功能。

## 数据源的重要声明

**注意**

这些只是声明。不要盲目复制此代码。请参见下面的实际用法示例：

```python
class PandasData(feed.DataBase):
    '''
    ``dataname`` 参数继承自 ``feed.DataBase`` 是 pandas DataFrame
    '''

    params = (
        # datetime 的可能值（必须始终存在）
        #  None : datetime 是 Pandas Dataframe 中的 "index"
        #  -1 : 自动检测位置或大小写相同的名称
        #  >= 0 : pandas dataframe 中列的数值索引
        #  string : pandas dataframe 中的列名（作为索引）
        ('datetime', None),

        # 下面是可能的值：
        #  None : 列不存在
        #  -1 : 自动检测位置或大小写相同的名称
        #  >= 0 : pandas dataframe 中列的数值索引
        #  string : pandas dataframe 中的列名（作为索引）
        ('open', -1),
        ('high', -1),
        ('low', -1),
        ('close', -1),
        ('volume', -1),
        ('openinterest', -1),
    )
```

上述 PandasData 类的片段展示了关键点：

- 在实例化时，类的 `dataname` 参数包含 Pandas Dataframe
- 该参数继承自基类 `feed.DataBase`
- 新参数具有 DataSeries 中常规字段的名称，并遵循以下约定：

  - `datetime` (默认: None)
    - None: datetime 是 Pandas Dataframe 中的“索引”
    - -1: 自动检测位置或大小写相同的名称
    - >= 0: pandas dataframe 中列的数值索引
    - string: pandas dataframe 中的列名（作为索引）

  - `open`、`high`、`low`、`close`、`volume`、`openinterest` (默认: -1)
    - None: 列不存在
    - -1: 自动检测位置或大小写相同的名称
    - >= 0: pandas dataframe 中列的数值索引
    - string: pandas dataframe 中的列名（作为索引）

一个小示例应能够加载经过 Pandas 解析的标准 2006 示例数据，而不是直接由 backtrader 解析。

运行示例代码以使用 CSV 数据中的现有“头”：

```sh
$ ./panda-test.py
--------------------------------------------------
               Open     High      Low    Close  Volume  OpenInterest
Date
2006-01-02  3578.73  3605.95  3578.73  3604.33       0             0
2006-01-03  3604.08  3638.42  3601.84  3614.34       0             0
2006-01-04  3615.23  3652.46  3615.23  3652.46       0             0
```

相同的代码，但告诉脚本跳过头：

```sh
$ ./panda-test.py --noheaders
--------------------------------------------------
                  1        2        3        4  5  6
0
2006-01-02  3578.73  3605.95  3578.73  3604.33  0  0
2006-01-03  3604.08  3638.42  3601.84  3614.34  0  0
2006-01-04  3615.23  3652.46  3615.23  3652.46  0  0
```

第二次运行时，使用 pandas.read_csv：
- 跳过第一行输入（skiprows 参数设置为 1）
- 不查找头行（header 参数设置为 None）

backtrader 对 Pandas 的支持尝试自动检测列名是否已被使用，否则使用数值索引，并相应地进行操作，尽量提供最佳匹配。

以下图表展示了成功的结果。Pandas Dataframe 已正确加载（在两种情况下均如此）。

![image](link-to-image)

示例代码：

```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import argparse

import backtrader as bt
import backtrader.feeds as btfeeds

import pandas


def runstrat():
    args = parse_args()

    # 创建 cerebro 实体
    cerebro = bt.Cerebro(stdstats=False)

    # 添加策略
    cerebro.addstrategy(bt.Strategy)

    # 获取 pandas dataframe
    datapath = ('../../datas/2006-day-001.txt')

    # 模拟在请求 noheaders 时不存在头行
    skiprows = 1 if args.noheaders else 0
    header = None if args.noheaders else 0

    dataframe = pandas.read_csv(datapath,
                                skiprows=skiprows,
                                header=header,
                                parse_dates=True,
                                index_col=0)

    if not args.noprint:
        print('--------------------------------------------------')
        print(dataframe)
        print('--------------------------------------------------')

    # 将其传递给 backtrader 数据源并添加到 cerebro
    data = bt.feeds.PandasData(dataname=dataframe)

    cerebro.adddata(data)

    # 运行所有内容
    cerebro.run()

    # 绘制结果
    cerebro.plot(style='bar')


def parse_args():
    parser = argparse.ArgumentParser(
        description='Pandas 测试脚本')

    parser.add_argument('--noheaders', action='store_true', default=False,
                        required=False,
                        help='不使用头行')

    parser.add_argument('--noprint', action='store_true', default=False,
                        help='打印 dataframe')

    return parser.parse_args()


if __name__ == '__main__':
    runstrat()
```
