---
title: "安装指南"
weight: 2
---

# 安装指南

本文将介绍如何安装和配置 Backtrader，以便您能快速开始交易策略的开发与测试。

## 需求和版本

Backtrader 是一个自包含的平台，无需外部依赖（除非您需要使用绘图功能）。

它的基本需求和兼容版本如下：

**基本需求：**

- Python 2.7
- Python 3.2 / 3.3 / 3.4 / 3.5
- pypy/pypy3

**额外需求（如需绘图）：**

- Matplotlib >= 1.4.1（虽然更早的版本可能也能工作，但这是开发时使用的版本）

**注意：** 在撰写本文时，Matplotlib 还不支持 pypy/pypy3。

## Python 2.x/3.x 兼容性

Backtrader 的开发主要在 Python 2.7 下进行，有时也会在 3.4 下进行测试。为了确保兼容性，本地测试环境会使用这两个版本。此外，通过 Travis 持续集成系统，我们还确保了与 Python 3.2 / 3.3 / 3.5 以及 pypy/pypy3 的兼容性。

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

## 从源码安装

1. 从 GitHub 网站下载发布版本或最新的 tarball：[下载链接](https://github.com/mementum/backtrader)
2. 解压下载的文件，然后运行以下命令安装：
```bash
python setup.py install
```

## 在项目中从源码运行

如果您希望在项目中直接运行 Backtrader 的源码，可以按照以下步骤操作：

1. 从 GitHub 网站下载发布版本或最新的 tarball：[下载链接](https://github.com/mementum/backtrader)
2. 将 backtrader 包目录复制到您的项目中。如在类 Unix 系统，可执行命令：
```bash
tar xzf backtrader.tgz
cd backtrader
cp -r backtrader project_directory
```
3. 请记住，如果需要绘图功能，还需手动安装 Matplotlib。

通过这些步骤，您将能够轻松安装和使用 Backtrader 平台进行量化交易的开发和测试。希望这些说明对您有所帮助，祝您在使用 Backtrader 时一切顺利！
