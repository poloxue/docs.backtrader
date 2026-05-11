---
title: "策略参数"
weight: 2
---

# 策略参数
  
策略通常都需要**参数**，在 `backtrader` 中，参数可作为类属性声明，通过元组或字典的形式定义。

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

无论使用元组还是字典，声明后均可通过 `self.params` 或 `self.p` 访问参数值。

```python
class MyStrategy(bt.Strategy):
    params = (('period', 20),)
    def __init__(self):
        sma = btind.SimpleMovingAverage(self.data, period=self.p.period)
```


这个例子中，`self.p.period` 获取 `period` 参数的值。


## 参数继承

在一个类中定义参数后，子类会自动继承。你可以在子类中重写参数的默认值。

```python
class BaseStrategy(bt.Strategy):
   params = (('period', 20),)

class MyStrategy(BaseStrategy):
   params = (('period', 30),)  # 重写父类的 period 参数
```

使用多重继承时，子类会继承所有父类的参数。如果多个父类定义了同名参数，子类使用继承列表中最后一个类的默认值。

