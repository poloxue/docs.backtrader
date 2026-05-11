---
title: "Pyfolio"
weight: 2
---

# PyFolio

注意，截至 2017-07-25，pyfolio 的 API 已更改，`create_full_tear_sheet` 不再接受 `gross_lev` 作为命名参数。因此，集成示例无法正常工作。

> 引用 pyfolio 主页面 http://quantopian.github.io/pyfolio/ 的内容：
> pyfolio 是一个由 Quantopian Inc. 开发的金融投资组合表现和风险分析 Python 库。它与开源回测库 Zipline 配合良好，现在也能很好地与 backtrader 配合。所需内容包括：
> - 需要 pyfolio
> - 以及它的依赖项（如 pandas、seaborn 等）

注意，在与 0.5.1 版本集成期间，需要更新依赖项，例如将 seaborn 从 0.7.0-dev 更新到 0.7.1，因为缺少 `swarmplot` 方法。

## 用法

将 `PyFolio` 分析器添加到 cerebro 中：

```python
cerebro.addanalyzer(bt.analyzers.PyFolio)
```

运行并检索第一个策略：

```python
strats = cerebro.run()
strat0 = strats[0]
```

使用你给分析器指定的名称或默认名称（pyfolio）来检索分析器。例如：

```python
pyfolio = strats.analyzers.getbyname('pyfolio')
```

使用分析器的 `get_pf_items` 方法检索 pyfolio 需要的四个组件：

```python
returns, positions, transactions, gross_lev = pyfolio.get_pf_items()
```

注意

集成是通过查看 `pyfolio` 的测试样本并复制相同的标题来实现的。

与 pyfolio 协同工作（已超出 backtrader 生态系统范围）

一些与 backtrader 无关的使用注意事项：

- pyfolio 自动绘图在 Jupyter Notebook 之外也能工作，但在 Notebook 内部效果最佳。
- pyfolio 数据表的输出在 Jupyter Notebook 之外几乎无法使用，但在 Notebook 内部可以正常工作。

结论很简单：如果希望使用 pyfolio，请在 Jupyter Notebook 中操作。

示例代码
代码看起来像这样：

```python
...
cerebro.addanalyzer(bt.analyzers.PyFolio, _name='pyfolio')
...
results = cerebro.run()
strat = results[0]
pyfoliozer = strat.analyzers.getbyname('pyfolio')
returns, positions, transactions, gross_lev = pyfoliozer.get_pf_items()
...
...
# pyfolio showtime
import pyfolio as pf
pf.create_full_tear_sheet(
    returns,
    positions=positions,
    transactions=transactions,
    gross_lev=gross_lev,
    live_start_date='2005-05-01',  # 此日期是样本特定的
    round_trips=True)

# 此时表格和图表将显示
```

## 参考

查看分析器参考以了解 PyFolio 分析器及其内部使用的分析器。
