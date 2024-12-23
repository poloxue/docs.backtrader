---
title: "异常 Exceptions"
weight: 4
---

# 异常

设计目标之一是尽早退出并让用户完全透明地了解错误发生的情况。目的是强制自己编写在异常情况下会中断的代码，并强制重新审视受影响的部分。

但现在是时候了，某些异常可能会慢慢添加到平台中。

#### 继承层次结构

所有异常的基类是 `BacktraderError`（直接继承自 `Exception`）。

#### 位置

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

#### 异常

- `StrategySkipError`
  - 请求平台跳过该策略的回测。应在实例的初始化（`__init__`）阶段引发。
