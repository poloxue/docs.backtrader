---
title: "策略参数"
weight: 2
---

# 策略参数
  
策略基本上都需要**参数**，而在 `backtrader` 中，这些参数可作为类属性进行声明。我们可以通过元组或字典的形式声明这些策略变量。

元组：

```python
class MyStrategy(bt.Strategy):
    params = (('period', 20),)
```

字典：

```python
class MyStrategy(bt.Strategy):
    params = dict(period=20)
```

无论是元组还是字典，参数声明后，都可以通过 `self.params` 或 `self.p` 访问参数的值。

```python
class MyStrategy(bt.Strategy):
    params = (('period', 20),)
    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.p.period)
```


在这个例子中，`self.p.period` 就是获取 `period` 参数的值。


## **参数继承**

如果你在一个类中定义了参数，子类会自动继承这些参数。你可以在子类中重写这些参数的默认值。

```python
class BaseStrategy(bt.Strategy):
   params = (('period', 20),)

class MyStrategy(BaseStrategy):
   params = (('period', 30),)  # 重写父类的 period 参数
```

如果你使用多重继承，子类会继承所有父类的参数。如果多个父类定义了相同的参数，子类会使用继承列表中最后一个类的默认值。

