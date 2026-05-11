---
title: "优化改进"
weight: 4
---

# 优化改进

**Backtrader** 通过引入两个新的 Cerebro 参数，优化了在多进程环境下管理数据源和结果的方式。

参数说明：

| 参数名   | 默认值 | 描述                                                               |
|----------|--------|--------------------------------------------------------------------|
| optdatas | True   | 如果为True并进行优化（系统可以预加载并使用runonce），则数据预加载将仅在主进程中进行一次，以节省时间和资源。 |
| optreturn | True   | 如果为True，优化结果将不是完整的策略对象（包括所有数据、指标、观察者等），而是带有以下属性的对象（与策略中相同）：<ul style="list-style-type: none; padding-left: 0;"><li>- params（或p）：策略执行时的参数</li><li>- analyzers：策略执行的分析器</li></ul> |

通常只需查看策略参数组合的最终表现（如收益率）。如需查看运行过程中的详细数据（如每个时间点的指标值），请关闭此选项。

## 数据源管理

在优化场景中，Cerebro 参数可能的组合是：

- `preload=True`（默认）：数据源会在执行任何回测代码前完成预加载。
- `runonce=True`（默认）：指标在紧凑的 for 循环中批量计算，而非逐步计算。

如果两个条件均为 True 且 `optdatas=True`，则数据源会在生成新子进程之前在主进程中预加载（子进程负责执行回测）。

## 结果管理

在优化场景中，评估每次策略运行的不同参数时，最重要的两个因素是：

`strategy.params`（或 `strategy.p`）

回测使用的实际参数集

`strategy.analyzers`

提供策略实际表现评估的对象。例如：`SharpeRatio_A`（年化夏普比率）

当 `optreturn=True` 时，不会返回完整的策略实例，而是创建占位符对象，仅携带上述两个属性供评估。

这避免了传回大量生成的数据，例如回测期间指标生成的值。

如需返回完整的策略对象，在实例化 Cerebro 或调用 `cerebro.run` 时设置 `optreturn=False` 即可。

## 一些测试运行

backtrader 源码中的优化示例已扩展并添加了对 `optdatas` 和 `optreturn` 的控制（实际上是禁用它们）。

### 单核心运行

作为参考，当将CPU数量限制为1且不使用多进程模块时会发生什么：

```sh
$ ./optimization.py --maxcpus 1
==================================================
**************************************************
--------------------------------------------------
OrderedDict([(u'smaperiod', 10), (u'macdperiod1', 12), (u'macdperiod2', 26), (u'macdperiod3', 9)])
**************************************************
--------------------------------------------------
OrderedDict([(u'smaperiod', 10), (u'macdperiod1', 13), (u'macdperiod2', 26), (u'macdperiod3', 9)])
...
...
OrderedDict([(u'smaperiod', 29), (u'macdperiod1', 19), (u'macdperiod2', 29), (u'macdperiod3', 14)])
==================================================
Time used: 184.922727833
```

### 多核心运行

在不限制 CPU 数量的情况下，Python 多进程模块会尝试使用所有 CPU。以下测试会禁用 `optdatas` 和 `optreturn`：

**`optdatas`和`optreturn`都启用**

默认行为：

```sh
$ ./optimization.py
...
...
...
==================================================
Time used: 56.5889185394
```

通过多核以及数据源和结果的优化，总时间从 184.92 秒减少到 56.58 秒。

请注意，示例使用了252条数据，并且指标仅生成长度为252点的值。这只是一个例子。

关键问题是这些改进有多少归功于新行为。

**`optreturn` 禁用**：将完整的策略对象传回调用者：

```sh
$ ./optimization.py --no-optreturn
...
...
...
==================================================
Time used: 67.056914007
```

执行时间增加了 18.50%（速度提升 15.62%）。

**`optdatas` 禁用**：每个子进程被迫加载各自的数据源值：

```sh
$ ./optimization.py --no-optdatas
...
...
...
==================================================
Time used: 72.7238112637
```

执行时间增加了 28.52%（速度提升 22.19%）。

**两者都禁用**：仍然使用多核，但使用旧的未改进行为：

```sh
$ ./optimization.py --no-optdatas --no-optreturn
...
...
...
==================================================
Time used: 83.6246643786
```

执行时间增加了 47.79%（速度提升 32.34%）。

这表明使用多个核心是时间改进的主要贡献因素。

**注意**：这些测试是在配备 i7-4710HQ（4 核/8 逻辑线程）和 16 GB RAM 的笔记本电脑上进行的，操作系统为 Windows 10 64 位。其他环境下结果可能有所不同。

## 总结

在优化过程中时间减少的最大因素是使用多个核心。

使用 `optdatas` 和 `optreturn` 的测试运行分别实现了约 22.19% 和 15.62% 的速度提升（两者结合则提升了 32.34%）。

## 示例使用

```sh
$ ./optimization.py --help
usage: optimization.py [-h] [--data DATA] [--fromdate FROMDATE]
                       [--todate TODATE] [--maxcpus MAXCPUS] [--no-runonce]
                       [--exactbars EXACTBARS] [--no-optdatas]
                       [--no-optreturn] [--ma_low MA_LOW] [--ma_high MA_HIGH]
                       [--m1_low M1_LOW] [--m1_high M1_HIGH] [--m2_low M2_LOW]
                       [--m2_high M2_HIGH] [--m3_low M3_LOW]
                       [--m3_high M3_HIGH]

Optimization

optional arguments:
  -h, --help            show this help message and exit
  --data DATA, -d DATA  data to add to the system
  --fromdate FROMDATE, -f FROMDATE
                        Starting date in YYYY-MM-DD format
  --todate TODATE, -t TODATE
                        Starting date in YYYY-MM-DD format
  --maxcpus MAXCPUS, -m MAXCPUS
                        Number of CPUs to use in the optimization
                          - 0 (default): use all available CPUs
                          - 1 -> n: use as many as specified
  --no-runonce          Run in next mode
  --exactbars EXACTBARS
                        Use the specified exactbars still compatible with preload
                          0 No memory savings
                          -1 Moderate memory savings
                          -2 Less moderate memory savings
  --no-optdatas         Do not optimize data preloading in optimization
  --no-optreturn        Do not optimize the returned values to save time
  --ma_low MA_LOW       SMA range low to optimize
  --ma_high MA_HIGH     SMA range high to optimize
  --m1_low M1_LOW       MACD Fast MA range low to optimize
  --m1_high M1_HIGH     MACD Fast MA range high to optimize
  --m2_low M2_LOW       MACD Slow MA range low to optimize
  --m2_high M2_HIGH     MACD Slow MA range high to optimize
  --m3_low M3_LOW       MACD Signal range low to optimize
  --m3_high M3_HIGH     MACD Signal range high to optimize
(C) 2015-2024 Daniel Rodriguez
```
