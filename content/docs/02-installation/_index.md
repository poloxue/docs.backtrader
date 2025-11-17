---
title: "安装指南"
weight: 2
---

# 安装指南

{{< youtube WxF3SAZ8US8 >}}

本文是 Backtrader 的安装指南，介绍如何安装 Backtrader。

## 需求和版本

官方仓库注明是 Backtrader 可运行在 Python 3.2-3.7，但它目前也能在 Python 3.10 及更新版本也可正常运行。如需绘图，请安装 Matplotlib >= 1.4.1；

## 兼容性

Backtrader 在 Python 2.x 和 3.x  上的兼容行如何呢？Backtrader 的开发主要在 Python 2.7 下进行，有时也会在 3.4 下进行测试。为了确保兼容性，本地测试环境会使用这两个版本。

当然，Backtrader 是支持最新的 Python3 版本的，我在 Python3.13 上测试也可正常运行。

## 从 PyPI 安装

首先，使用 pip 轻松安装 Backtrader：

```bash
pip install backtrader
```

**如果需要绘图功能：**

```bash
pip install "backtrader[plotting]"
```

这会自动安装 Matplotlib 及其依赖项。

## 测试安装

测试代码 buyhold.py，如下所示：

```python
import yfinance as yf
import backtrader as bt
from datetime import datetime


class BuyHold(bt.Strategy):
    def next(self):
        if not self.position:
            self.buy()


cerebro = bt.Cerebro()
cerebro.addstrategy(BuyHold)

raw_data = yf.download(
    "AAPL", start="2020-01-01", end="2021-01-01", multi_level_index=False
)

apple_data = bt.feeds.PandasData(dataname=raw_data)
cerebro.adddata(apple_data)
cerebro.broker.setcash(1e8)
print(f"最初资金: {cerebro.broker.getcash()}")
cerebro.run()
print(f"最终资金: {cerebro.broker.getcash()}")
cerebro.plot()
```

为了正常使用这个代码，除了 backtrader，还要安装 yfinance，便于从远程下载行情数据。

```bash
python buyhold.py
```

如果安装成功，将能输出初始最终的资金。

## 源码安装

从 GitHub 通过 git 下载发布最新版本，访问 [GitHub 仓库地址](https://github.com/mementum/backtrader)。你可以将 Backtrader 直接包含在你的项目源码中，可从 GitHub 下载发布版本，并将 backtrader 复制到您的项目中。

```bash
git clone https://github.com:mementum/backtrader src
cp -r src/backtrader project_directory/
cd project_directory
export PYTHONPATH=`pwd`
```

但请记住，如果需要绘图功能，还需手动安装 Matplotlib。

这种运行方式的最大好处是，开发策略时，也能非常方便的调试和阅读 Backtrader 的核心源码。

## 最后

如果你追求更快的回测速度，可尝试使用 PyPy3 环境运行，但需注意它的绘图支持较弱。
