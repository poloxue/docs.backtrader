---
title: "Pyfolio"
weight: 2
---

# PyFolio

注意，截至（至少）2017-07-25，pyfolio 的 API 已更改，`create_full_tear_sheet` 不再有 `gross_lev` 作为命名参数。因此，集成示例无法正常工作。

> 引用 pyfolio 主页面 http://quantopian.github.io/pyfolio/ 的内容：
> pyfolio 是一个由 Quantopian Inc. 开发的用于金融投资组合的表现和风险分析的 Python 库。它与开源回测库 Zipline 配合良好，现在它也可以很好地与 backtrader 配合。所需的内容包括：
> - 显然需要 pyfolio
> - 以及它的依赖项（如 pandas、seaborn 等）

注意，在与 0.5.1 版本集成期间，需要更新依赖项的最新版本，例如将之前安装的 seaborn 从 0.7.0-dev 更新到 0.7.1，显然是由于缺少方法 `swarmplot`。

## 用法

将 PyFolio 分析器添加到 cerebro 中：

```python
cerebro.addanalyzer(bt.analyzers.PyFolio)
```

运行并检索第一个策略：

```python
strats = cerebro.run()
strat0 = strats[0]
```

使用你给分析器命名的名称或默认名称（pyfolio）来检索分析器。例如：

```python
pyfolio = strats.analyzers.getbyname('pyfolio')
```

使用分析器方法 `get_pf_items` 检索 pyfolio 后续需要的四个组件：

```python
returns, positions, transactions, gross_lev = pyfolio.get_pf_items()
```

注意

集成是通过查看 `pyfolio` 的测试样本并复制相同的标题（或缺少的部分）来完成的。

与 pyfolio 一起工作（这已经超出了 backtrader 生态系统的范围）

一些与 backtrader 无关的使用注意事项：

- pyfolio 自动绘图在 Jupyter Notebook 之外也能工作，但在 Notebook 内部效果最好。
- pyfolio 数据表的输出在 Jupyter Notebook 之外几乎不起作用，但在 Notebook 内部可以正常工作。

结论很简单，如果希望使用 pyfolio：在 Jupyter Notebook 内工作。

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
