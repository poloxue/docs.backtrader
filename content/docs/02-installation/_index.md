---
title: "安装指南"
weight: 2
---

# 安装指南

{{< youtube WxF3SAZ8US8 >}}

本文是 Backtrader 量化交易框架的安装指南。我们将详细介绍多种安装方式，并涵盖版本兼容性、功能依赖和验证安装的完整流程。

## 环境与兼容性
Backtrader 官方支持 Python 3.2-3.7，但实际测试表明，它在 Python 3.10 乃至最新的 3.13 版本上都能稳定运行。

虽然官方主要基于 Python 2.7 进行开发，并在 3.4 环境下测试，但当前版本对新版 Python 的兼容性表现良好。如需使用绘图功能，请确保安装 Matplotlib 1.4.1 或更高版本。

## 安装方式

最便捷的方式是通过 PyPI 进行安装。只需执行以下命令即可完成基础安装：
```bash
pip install backtrader
```
如果需要使用绘图功能，建议安装包含 Matplotlib 的完整版本：
```bash
pip install "backtrader[plotting]"
```
这条命令会自动安装 Matplotlib 及其所有依赖项，省去手动配置的麻烦。

## 验证安装效果

为了确认安装成功，我们可以运行一个简单的测试策略。以下是完整的测试代码示例：
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
print(f"初始资金: {cerebro.broker.getcash()}")
cerebro.run()
print(f"最终资金: {cerebro.broker.getcash()}")
cerebro.plot()
```

运行前请确保已安装 yfinance 库：

```bash
pip install yfinance
```

执行脚本：

```bash
python buyhold.py
```

如果能看到资金变化并显示交易图表，说明安装完全成功。

## 源码安装方式

对于希望深入研究的用户，可以从 GitHub 获取最新源码：
```bash
git clone https://github.com:mementum/backtrader src
cp -r src/backtrader project_directory/
cd project_directory
export PYTHONPATH=`pwd`
```
这种方式虽然需要手动安装 Matplotlib，但便于调试和阅读核心代码，特别适合需要进行深度定制的用户。

## 额外提示

如果追求极致的回测性能，可以尝试在 PyPy3 环境下运行 Backtrader，不过需要注意其绘图功能支持相对有限。无论选择哪种安装方式，都建议通过测试代码验证各项功能是否正常。
