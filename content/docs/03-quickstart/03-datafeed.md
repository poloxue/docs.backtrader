---
title: "配置数据"
weight: 3
---

# 配置数据

本文具体讲讲如何在 Backtrader 中配置数据源，也就是 DataFeed。

我会先演示如何给 Backtrader 添加 CSV 数据源。然后，再介绍 Backtrader 中最常用的两种 DataFeed 的使用。

## 添加数据

首先，来演示给 Backtrader 添加数据源吧。

给 Backtrader 加载数据源的过程可分为两个步骤，首先是创建 DataFeed，然后就是将 DataFeed 加载到 cerebro。

```python
data = bt.feeds.{DataFeedClass}
cerebro.adddata(data)
```

Backtrader 提供了种类繁多的 DataFeedClass，可用于对接处理不同的数据源，如 CSV 文件、Pandas DataFrame 或者其他自定义的数据类。

下面来个具体的案例，先用 CSV 数据文件作为演示案例。

先从 Backtrader 代码仓库中下载了 Oracle 从 1995 年到 2014 的行情数据文件[orcl-1995-2014.csv](https://github.com/mementum/backtrader/blob/master/datas/orcl-1995-2014.txt)。

开始编写代码吧。

首先是读取 csv 数据文件创建 DataFeed。Backtrader 内置的 `bt.feeds.GenericCSVData` 类来读取 CSV 数据文件。

具体操作代码如下：

```python
data = bt.feeds.GenericCSVData(
    dataname="./orcl-1995-2014.txt",
    dtformat="%Y-%m-%d",
)
```

指定数据文件的路径和设置日期格式。

GenericCSVData 能适应各种格式的 CSV 文件，是一个通用的读取 CSV 行情数据的 DataFeed 类。

Datafeed 创建好后，接下来的关键一步，即通过 cerebro.adddata(data) 把数据加载到回测引擎中。

```python
cerebro.adddata(data)
```

这个脚本的代码：

```python
import backtrader as bt


def main():
    cerebro = bt.Cerebro()

    data = bt.feeds.GenericCSVData(
        dataname="./orcl-1995-2014.txt", dtformat="%Y-%m-%d",
    )
    cerebro.adddata(data)

    cerebro.run()
    cerebro.plot()


if __name__ == "__main__":
    main()
```

运行这个代码，现在就能看到输出的图表中是有价格行情了。

![](https://cdn.jsdelivr.net/gh/poloxue/images@backtrader/03-quickstart-03datafeed-01.png)

## 创建数据

前面是通过从远程下载 CSV 文件，通过 Backtrader 的 GenericCSVData 读取 CSV 文件创建 DataFeed。Backtrader 还支持数据获取方式。

这里就再介绍一种更常用的 DataFeed 数据创建方式：基于 Pandas 的 DataFrame 创建 DataFeed。

依赖 pandas 的灵活性，这种方式能极大扩展你的数据渠道，无论是本地文件，如 CSV 文件，还是关系型数据，如 CSV，或是远程 API 下载，如 yfinance、tushare、akshare 等，都能搞定。

好，我们直接来看代码。

现在通过 yfinance 下载行情数据，而不是手动下载 CSV 文件再加载。

下载和加载 BTC-USD 从 2024-01-01 到 2025-11-10 的行情数据

```python
cerebro = bt.Cerebro()
df = yf.download("BTC-USD", start="2024-01-01", end='2025-11-10', multi_level_index=False)
data = bt.feeds.PandasData(dataname=df)
cerebro.adddata(data)
cerebro.run()
cerebro.plot(style='bar')
```

注，记得使用最新的 yfinance 要设置 multi_level_index=False，否则可能得到多级索引的数据，导致加载失败。

输出绘图如下所示：

![](https://cdn.jsdelivr.net/gh/poloxue/images@backtrader/03-quickstart-03datafeed-02.png)

如上所示，我们先用 yfinance 获取函数下载指定品种和时间段的数据，得到一个 DataFrame。然后传递给 PandasData 就可创建出满足 Backtrader 的数据了。

实际上 PandasData 对传入的数据是有一定要求的。

索引是时间，数据列要满足 open，high，low，close，volume等。当然，数据列名是大写，如 Open、High、Low、Close 也是可以的。PandasData 内部会自动转化为小写。

准备工作做完后，个 DataFrame 就可用 bt.feeds.PandasData 来包装了。然后通过 cerebro.adddata() 把它添加到回测系统中。

好了，现在我们的回测系统里已经有了数据。

## 最后说明

Backtrader 的数据能力是非常强大，本文只是简单的介绍创建和添加数据。除此，Backtrader 还提供了其他丰富多样的 DataFeed 类，当然大部分情况，本文的 GenericCSVData 和 PandasData 已经基本能满足大部分的情况。如果有自定义数据列的需求，Backtrader 也是只是自定义数据的。

添加数据部分，除了 `cerebro.adddata` 直接将数据添加到回测系统，还可以重放、重采样数据，模拟实际交易环境，或者多周期交易。

下一节，我们学习如何在 Backtrader 中编写我们的交易策略。

