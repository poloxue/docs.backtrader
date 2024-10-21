---
title: "配置数据"
weight: 3
---

# 配置数据 DataFeed

我们目标是通过策略实现资产增值，这就离不开价格数据，甚至是其他有用的数据。本小节，我们将学习如何系统配置数据源，即添加数据源 DataFeed。

配置 DataFeed 要用到的是 `backtrader.feeds` 中提供的数据工具。要用到的数据文件是 [orcl-1995-2014](https://raw.githubusercontent.com/mementum/backtrader/master/datas/orcl-1995-2014.txt)（点击下载即可下载）。

假设，数据文件被下载到当前目录，通过 `bt.feeds.YahooFinanceCSVDataFeed` 即可创建 datafeed。

```python
data = bt.feeds.YahooFinanceCSVData(
  dataname="./orcl-1995-2014.txt",
  fromdate=datetime.datetime(2000, 1, 1),
  todate=datetime.datetime(2000, 12, 31),
  reverse=False
)
```

Yahoo 在线下载的 CSV 数据按日期降序排列，YahooFinanceCSVData 也是按这个标准解析。但我们提供的数据是升序排列，故设置参数 `reverse=True`。

接下来，我们通过 `cerebro.adddata` 将数据添加系统即可。

```python
cerebro.adddata(data)
```

将这部分实现补充到我们的系统中。

## 完整示例

```python
import datetime  # For datetime objects
import os.path  # To manage paths
import sys  # To find out the script name (in argv[0])
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()

    modpath = os.path.dirname(os.path.abspath(sys.argv[0]))
    datapath = os.path.join(modpath, './datas/orcl-1995-2014.txt')

    data = bt.feeds.YahooFinanceCSVData(
        dataname=datapath,
        fromdate=datetime.datetime(2000, 1, 1),
        todate=datetime.datetime(2000, 12, 31),
        reverse=False)

    cerebro.adddata(data)
    cerebro.broker.setcash(100000.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
```

运行输出：

```
Starting Portfolio Value: 100000.00
Final Portfolio Value: 100000.00
```

系统已经有了数据源，但资产并未增值，为什么？让我们继续下一节，策略（Strategy）类的学习。


