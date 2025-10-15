---
title: "安装指南"
weight: 2
---

# 安装指南

本文是 Backtrader 的安装指南，介绍如何安装 Backtrader。

## 需求和版本

Backtrader 注明是可运行在 Pyhon3.2-3.7，但它目前也能在 Python 3.10 及以上版本中正常运行。如需绘图，请安装 Matplotlib >= 1.4.1；

如果你追求更快的回测速度，可尝试使用 PyPy3 环境运行，但需注意它的绘图支持较弱。

## 兼容性

Backtrader 在 Python 2.x 和 3.x  上的兼容行如何呢？

Backtrader 的开发主要在 Python 2.7 下进行，有时也会在 3.4 下进行测试。为了确保兼容性，本地测试环境会使用这两个版本。

开发过程中，通过 Travis 持续集成系统，还确保了与 Python 3.2 / 3.3 / 3.5 以及 pypy/pypy3 的兼容性。

## 从 PyPI 安装

您可以使用 pip 轻松安装 Backtrader：
```bash
pip install backtrader
```

或者，您也可以使用 easy_install 安装：
```bash
easy_install backtrader
```

**如果需要绘图功能：**
```bash
pip install "backtrader[plotting]"
```
这会自动安装 Matplotlib 及其依赖项。

## 源码安装

从 GitHub 通过 git 下载发布最新版本，访问 [GitHub 仓库地址](https://github.com/mementum/backtrader)。

运行命令安装：

```bash
git clone https://github.com:mementum/backtrader
cd backtrader
python setup.py install
```

## 源码运行

如果你希望将 Backtrader 直接包含在你的项目源码中，可从 GitHub 下载发布版本，并将 backtrader 复制到您的项目中。

```bash
git clone https://github.com:mementum/backtrader
cp -r backtrader project_directory
```

> 请记住，如果需要绘图功能，还需手动安装 Matplotlib。

这种运行方式的最大好处是，开发策略时，也能非常方便的调试和阅读 Backtrader 的核心源码。
