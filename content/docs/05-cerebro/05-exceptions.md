---
title: "异常 Exceptions"
weight: 5
---

# 异常

设计目标之一是尽早退出，让用户清楚地了解错误发生的位置。这迫使我们在异常情况下编写会中断的代码，从而必须重新审视受影响的部分。

但时机已到，平台开始逐步引入一些异常类型。

## 继承层次结构

所有异常的基类是 `BacktraderError`（直接继承自 `Exception`）。

## 位置

在 `errors` 模块内，可以通过以下方式访问：

```python
import backtrader as bt

class Strategy(bt.Strategy):

    def __init__(self):
        if something_goes_wrong():
            raise bt.errors.StrategySkipError
```

或者直接从 `backtrader` 访问：

```python
import backtrader as bt

class Strategy(bt.Strategy):

    def __init__(self):
        if something_goes_wrong():
            raise bt.StrategySkipError
```

## 异常

`StrategySkipError`，请求平台跳过该策略的回测。应在策略实例的 `__init__` 阶段抛出。
