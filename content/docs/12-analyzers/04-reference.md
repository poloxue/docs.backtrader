---
title: "Reference"
weight: 4
---

# 参考

## AnnualReturn

```python
class backtrader.analyzers.AnnualReturn()
```

该分析器通过查看年的起点和终点来计算年度回报率。

**参数：**
- 无

**成员属性：**
- `rets`：计算出的年度回报率列表
- `ret`：年度回报率字典（键：年份）

**get_analysis：**
返回包含年度回报率的字典（键：年份）

---

## Calmar

```python
class backtrader.analyzers.Calmar()
```

该分析器计算 Calmar 比率，时间框架可以与基础数据使用的不同。

**参数：**
- `timeframe`（默认：无）：如果为 None，将使用系统中第一个数据的时间框架。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。

**get_analysis：**
返回一个包含时间段键和对应滚动 Calmar 比率的 OrderedDict。

---

## DrawDown

```python
class backtrader.analyzers.DrawDown()
```

该分析器计算交易系统的回撤统计数据，如百分比和美元的回撤值、最大回撤值、回撤长度和最大回撤长度。

**参数：**
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。

**get_analysis：**
返回一个包含回撤统计数据的字典（支持 . 符号表示法和子字典），可用的键/属性包括：
- `drawdown`：回撤值（百分比）
- `moneydown`：回撤值（货币单位）
- `len`：回撤长度
- `max.drawdown`：最大回撤值（百分比）
- `max.moneydown`：最大回撤值（货币单位）
- `max.len`：最大回撤长度

---

## TimeDrawDown

```python
class backtrader.analyzers.TimeDrawDown()
```

该分析器计算在选定时间框架上的交易系统回撤，可以与基础数据使用的时间框架不同。

**参数：**
- `timeframe`（默认：无）：如果为 None，将使用系统中第一个数据的时间框架。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。

**get_analysis：**
返回一个包含回撤统计数据的字典（支持 . 符号表示法和子字典），可用的键/属性包括：
- `drawdown`：回撤值（百分比）
- `maxdrawdown`：最大回撤值（货币单位）
- `maxdrawdownperiod`：回撤长度

---

## GrossLeverage

```python
class backtrader.analyzers.GrossLeverage()
```

该分析器计算当前策略的总杠杆。

**参数：**
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。

**get_analysis：**
返回一个字典，其中返回值为值，每个返回值的日期时间点为键。

---

## PositionsValue

```python
class backtrader.analyzers.PositionsValue()
```

该分析器报告当前数据集的仓位价值。

**参数：**
- `timeframe`（默认：无）：如果为 None，将使用系统中第一个数据的时间框架。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `headers`（默认：False）：在保存结果的字典中添加一个初始键，名称为数据的名称（'Datetime' 为键）。
- `cash`（默认：False）：包括实际现金作为额外仓位（对于标头，将使用名称 'cash'）。

**get_analysis：**
返回一个字典，其中返回值为值，每个返回值的日期时间点为键。

---

## PyFolio

```python
class backtrader.analyzers.PyFolio()
```

该分析器使用 4 个子分析器收集数据，并将其转换为与 pyfolio 兼容的数据集。

**子分析器：**
- `TimeReturn`：用于计算全球投资组合价值的回报。
- `PositionsValue`：用于计算每个数据的仓位价值。设置 `headers` 和 `cash` 参数为 True。
- `Transactions`：用于记录每个数据上的每笔交易（数量、价格、价值）。设置 `headers` 参数为 True。
- `GrossLeverage`：跟踪总杠杆（策略投资的程度）。

**参数：**
这些参数透明地传递给子分析器。

- `timeframe`（默认：`bt.TimeFrame.Days`）：如果为 None，将使用系统中第一个数据的时间框架。
- `compression`（默认：1）：如果为 None，将使用系统中第一个数据的压缩。

**get_analysis：**
返回一个字典，其中返回值为值，每个返回值的日期时间点为键。

**get_pf_items：**
返回一个包含 4 个元素的元组，可用于进一步处理 `pyfolio`。

- `returns`，`positions`，`transactions`，`gross_leverage`

由于这些对象旨在作为 pyfolio 的直接输入，因此此方法会本地导入 pandas，将内部 backtrader 结果转换为 pandas DataFrames，这是例如 pyfolio.create_full_tear_sheet 预期的输入。如果未安装 pandas，该方法将失败。

---

## LogReturnsRolling

```python
class backtrader.analyzers.LogReturnsRolling()
```

该分析器计算给定时间框架和压缩的滚动回报。

**参数：**
- `timeframe`（默认：无）：如果为 None，将使用系统中第一个数据的时间框架。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `data`（默认：无）：跟踪的参考资产，而不是投资组合价值。

**get_analysis：**
返回一个字典，其中返回值为值，每个返回值的日期时间点为键。

---

## PeriodStats

```python
class backtrader.analyzers.PeriodStats()
```

计算给定时间框架的基本统计数据。

**参数：**
- `timeframe`（默认：Years）：如果为 None，将使用系统中第一个数据的时间框架。
- `compression`（默认：1）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。

**get_analysis：**
返回一个包含以下键的字典：
- `average`
- `stddev`
- `positive`
- `negative`
- `nochange`
- `best`
- `worst`

如果参数 `zeroispos` 设置为 True，则没有变化的周期将计为正数。

---

## Returns

```python
class backtrader.analyzers.Returns()
```

使用对数方法计算总回报、平均回报、复合回报和年化回报。

**参数：**
- `timeframe`（默认：无）：如果为 None，将使用系统中第一个数据的时间框架。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `tann`（默认：无）：用于年化（归一化）回报的周期数：
  - 天：252
  - 周：52
  - 月：12
  - 年：1
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。

**get_analysis：**
返回一个字典，其中返回值为值，每个返回值的日期

时间点为键。返回的字典包含以下键：
- `rtot`：总复合回报
- `ravg`：整个期间的平均回报（特定时间框架）
- `rnorm`：年化/归一化回报
- `rnorm100`：以 100% 表示的年化/归一化回报

---

## SharpeRatio

```python
class backtrader.analyzers.SharpeRatio()
```

该分析器使用风险资产（即利率）计算策略的夏普比率。

**参数：**
- `timeframe`（默认：TimeFrame.Years）
- `compression`（默认：1）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `riskfreerate`（默认：0.01 -> 1%）：以年利率表示（见下文 `convertrate`）。
- `convertrate`（默认：True）：将年利率转换为月、周或日利率。不支持子日转换。
- `factor`（默认：无）：如果为 None，将从预定义表中选择年到所选时间框架的转换因子。天：252，周：52，月：12，年：1。否则将使用指定的值。
- `annualize`（默认：False）：如果 `convertrate` 为 True，夏普比率将在所选时间框架内提供。在大多数情况下，夏普比率以年化形式提供。
- `stddev_sample`（默认：False）：如果设置为 True，将在均值中减少分母 1 来计算标准差。这在计算标准差时使用，如果认为并非所有样本都用于计算。这被称为贝塞尔修正。
- `daysfactor`（默认：无）：旧命名为因子。如果设置为除 None 之外的任何值，并且时间框架为 TimeFrame.Days，将假设这是旧代码并使用该值。
- `legacyannual`（默认：False）：使用 AnnualReturn 分析器，顾名思义仅适用于年份。
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。

**get_analysis：**
返回一个包含键 “sharperatio” 的字典，其中包含比率。

---

## SharpeRatio_A

```python
class backtrader.analyzers.SharpeRatio_A()
```

夏普比率的扩展，直接以年化形式返回夏普比率。

**更改的参数：**
- `annualize`（默认：True）

---

## SQN

```python
class backtrader.analyzers.SQN()
```

SQN 或系统质量数。由 Van K. Tharp 定义，用于分类交易系统。

1.6 - 1.9 低于平均水平
2.0 - 2.4 平均水平
2.5 - 2.9 良好
3.0 - 5.0 优秀
5.1 - 6.9 杰出
7.0 - 圣杯？

公式：
```
SquareRoot(NumberTrades) * Average(TradesProfit) / StdDev(TradesProfit)
```
当交易数量 >= 30 时，sqn 值应被认为是可靠的。

**get_analysis：**
返回一个包含键 “sqn” 和 “trades” 的字典（已考虑的交易数量）。

---

## TimeReturn

```python
class backtrader.analyzers.TimeReturn()
```

该分析器通过查看时间框架的起点和终点来计算回报。

**参数：**
- `timeframe`（默认：无）：如果为 None，将使用系统中第一个数据的时间框架。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `data`（默认：无）：跟踪的参考资产，而不是投资组合价值。

**get_analysis：**
返回一个字典，其中返回值为值，每个返回值的日期时间点为键。

---

## TradeAnalyzer

```python
class backtrader.analyzers.TradeAnalyzer()
```

提供已平仓交易的统计数据（还保持未平仓交易的计数）。

- 总开仓/平仓交易
- 连胜/连败 当前/最长
- 总损益/平均损益
- 胜/负 计数/总损益/平均损益/最大损益
- 多/空 计数/总损益/平均损益/最大损益
- 胜/负 计数/总损益/平均损益/最大损益

**注意：**
分析器使用“自动”字典字段，这意味着如果没有执行交易，则不会生成统计数据。在这种情况下，`get_analysis` 返回的字典中将有一个单独的字段/子字段：

```
dictname[‘total’][‘total’] 将具有值 0（该字段也可以使用点符号 dictname.total.total 访问）。
```

---

## Transactions

```python
class backtrader.analyzers.Transactions()
```

该分析器报告系统中每个数据的交易情况。

**参数：**
- `headers`（默认：True）：在保存结果的字典中添加一个初始键，名称为数据的名称。

该分析器旨在便于与 pyfolio 集成，并从用于它的样本中获取标题名称：

- 'date', 'amount', 'price', 'sid', 'symbol', 'value'

**get_analysis：**
返回一个字典，其中返回值为值，每个返回值的日期时间点为键。

---

## VWR

```python
class backtrader.analyzers.VWR()
```

可变性加权回报：使用对数回报的更好的夏普比率。

**别名：**
- VariabilityWeightedReturn

**参数：**
- `timeframe`（默认：无）：如果为 None，则整个回测期间的回报将被报告。
- `compression`（默认：无）：仅用于子日时间框架，例如通过指定 "TimeFrame.Minutes" 和 60 作为压缩在每小时时间框架上工作。
- `tann`（默认：无）：用于年化（归一化）平均回报的周期数。如果为 None，则使用标准 t 值，即：
  - 天：252
  - 周：52
  - 月：12
  - 年：1
- `tau`（默认：2.0）：计算因子（见文献）
- `sdev_max`（默认：0.20）：最大标准差（见文献）
- `fund`（默认：无）：如果为 None，将自动检测经纪人的实际模式（fundmode - True/False）来决定回报率是基于总净资产价值还是基金价值。见经纪人文档中的 `set_fundmode`。

**get_analysis：**
返回一个字典，其中返回值为值，每个返回值的日期时间点为键。返回的字典包含以下键：
- `vwr`：可变性加权回报
