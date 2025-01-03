---
title: "MFI 通用版"
weight: 6
---

### MFI 通用版

在之前的《Canonical 与 Non-Canonical》一文中，介绍了 MFI（即 Money Flow Indicator，资金流动指标）的实现。

虽然这个实现是按照传统的方式开发的，但它依然有改进的空间，并且可以做得更通用。

让我们关注一下实现的前几行代码，即计算典型价格的部分。

#### Canonical MFI - 典型价格和原始资金流

```python
class MFI_Canonical(bt.Indicator):
    lines = ('mfi',)
    params = dict(period=14)

    def __init__(self):
        tprice = (self.data.close + self.data.low + self.data.high) / 3.0
        mfraw = tprice * self.data.volume
        ...
```

典型的实例化方式如下所示：

#### MFI 典型实例化

```python
class MyMFIStrategy(bt.Strategy):

    def __init__(self):
        mfi = bt.MFI_Canonical(self.data)
```

这里的问题应该很明显： “需要为指标提供包含收盘价、最低价、最高价和成交量的输入（即*在 backtrader 生态系统中的*lines）”。

当然，也可能有人希望使用来自不同数据源的组件来创建资金流动指标（例如来自数据源或其他指标的线）。例如，想要给收盘价赋予更多权重，而不需要开发一个特定的指标。考虑到行业标准的 OHLCV 数据字段顺序，支持多个输入并给收盘价加权的实例化可以是这样的：

#### MFI 多输入实例化

```python
class MyMFIStrategy2(bt.Strategy):

    def __init__(self):
        wclose = self.data.close * 5.0
        mfi = bt.MFI_Canonical(self.data.high, self.data.low,
                               wclose, self.data.volume)
```

或者因为用户之前使用过 ta-lib 并喜欢多个输入的方式。

#### 支持多个输入

backtrader 尽量做到符合 Python 风格，并且系统中的 `self.datas` 数组（自动提供给策略的所有数据源）可以查询其长度。我们可以利用这一点来区分调用者的需求，并正确地计算 `tprice` 和 `mfraw`。

#### MFI - 使用 `len` 的多个输入

```python
class MFI_MultipleInputs(bt.Indicator):
    lines = ('mfi',)
    params = dict(period=14)

    def __init__(self):
        if len(self.datas) == 1:
            # 传入一个数据源，必须包含各个组件
            tprice = (self.data.close + self.data.low + self.data.high) / 3.0
            mfraw = tprice * self.data.volume
        else:
            # 如果有多个数据源，按照 OHLCV 顺序传入每个组件
            tprice = (self.data0 + self.data1 + self.data2) / 3.0
            mfraw = tprice * self.data3

        # 与之前的实现无变化
        flowpos = bt.ind.SumN(mfraw * (tprice > tprice(-1)), period=self.p.period)
        flowneg = bt.ind.SumN(mfraw * (tprice < tprice(-1)), period=self.p.period)

        mfiratio = bt.ind.DivByZero(flowpos, flowneg, zero=100.0)
        self.l.mfi = 100.0 - 100.0 / (1.0 + mfiratio)
```

#### 注意

注意如何引用各个组件，如 `self.dataX`（如 `self.data0`、`self.data1`）。

这与使用 `self.datas[x]`（如 `self.datas[0]`）是一样的。

接下来，让我们通过图形化展示该指标与传统实现的结果是否一致，尤其是在多个输入与数据源的原始组件相对应时。为此，我们将在策略中运行如下代码：

#### MFI - 结果检查

```python
class MyMFIStrategy2(bt.Strategy):

    def __init__(self):
        MFI_Canonical(self.data)
        MFI_MultipleInputs(self.data, plotname='MFI 单输入')
        MFI_MultipleInputs(self.data.high,
                           self.data.low,
                           self.data.close,
                           self.data.volume,
                           plotname='MFI 多输入')
```

#### MFI 结果检查

通过图示来看，三者的结果应该是相同的，而无需逐一检查每个值。

最后，让我们看看如果加大收盘价的权重，会发生什么。我们这样运行：

#### MFI - 5 倍收盘价

```python
class MyMFIStrategy2(bt.Strategy):
    def __init__(self):
        MFI_MultipleInputs(self.data)
        MFI_MultipleInputs(self.data.high,
                           self.data.low,
                           self.data.close * 5.0,
                           self.data.volume,
                           plotname='MFI 收盘价 * 5.0')
```

#### MFI 收盘价 * 5.0

是否合理留给读者自行判断，但可以清楚地看到，通过增加收盘价的权重，图形模式发生了变化。

### 结论

通过简单地使用 Pythonic 风格的 `len`，我们可以将一个使用固定组件名称的数据源的指标转换为一个接受多个通用输入的指标。
