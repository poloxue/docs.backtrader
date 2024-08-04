---
title: "安装指南"
weight: 2
---

# 安装指南

## 需求和版本

**backtrader** 是一个自包含的平台，无需外部依赖（除非您需要绘图功能）。

**基本需求：**
- Python 2.7
- Python 3.2 / 3.3 / 3.4 / 3.5
- pypy/pypy3

**额外需求（如果需要绘图）：**
- Matplotlib >= 1.4.1（可能在更早的版本中工作，但这是开发时使用的版本）

**注意：** 撰写本文时，Matplotlib 不支持 pypy/pypy3。

## Python 2.x/3.x 兼容性

- 开发在 Python 2.7 下进行，有时也在 3.4 下进行。
- 测试在本地用这两个版本运行。
- 与 3.2 / 3.3 / 3.5 和 pypy/pypy3 的兼容性通过 Travis 的持续集成进行检查。

## 从 pypi 安装

使用 pip：
```bash
pip install backtrader
```
easy_install 也可以用相同的语法：

```bash
easy_install backtrader
```

**如果需要绘图功能：**
```bash
pip install "backtrader[plotting]"
```
这会拉取 matplotlib 和其他依赖项。

## 从源码安装

1. 从 GitHub 网站下载发布版本或最新的 tarball：[下载链接](https://github.com/mementum/backtrader)
2. 解压后运行以下命令：
```bash
python setup.py install
```

## 在项目中从源码运行

1. 从 GitHub 网站下载发布版本或最新的 tarball：[下载链接](https://github.com/mementum/backtrader)
2. 将 backtrader 包目录复制到您自己的项目中。例如，在类似 Unix 的操作系统下：
```bash
tar xzf backtrader.tgz
cd backtrader
cp -r backtrader project_directory
```
3. 请记住，您还需要手动安装 matplotlib 以进行绘图。

希望这些说明能帮助您轻松安装和使用 backtrader 平台！
