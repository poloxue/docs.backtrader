---
title: "参考"
weight: 2
---

### Sizers 参考

#### FixedSize

```python
class backtrader.sizers.FixedSize()
```

此 Sizer 仅为任何操作返回固定大小。通过指定 `tranches` 参数，可以控制系统希望用于逐步进入交易的批次数量。

**参数：**

- `stake`（默认：`1`）
- `tranches`（默认：`1`）

#### FixedReverser

```python
class backtrader.sizers.FixedReverser()
```

此 Sizer 返回反转已开头寸所需的固定大小或开仓所需的固定大小。

- 开仓：返回参数 `stake`
- 反转头寸：返回 `2 * stake`

**参数：**

- `stake`（默认：`1`）

#### PercentSizer

```python
class backtrader.sizers.PercentSizer()
```

此 Sizer 返回可用现金的百分比。

**参数：**

- `percents`（默认：`20`）

#### AllInSizer

```python
class backtrader.sizers.AllInSizer()
```

此 Sizer 返回经纪人所有可用现金。

**参数：**

- `percents`（默认：`100`）

#### PercentSizerInt

```python
class backtrader.sizers.PercentSizerInt()
```

此 Sizer 以整数形式返回可用现金的百分比，并将大小截断为整数。

**参数：**

- `percents`（默认：`20`）

#### AllInSizerInt

```python
class backtrader.sizers.AllInSizerInt()
```

此 Sizer 返回经纪人所有可用现金，并将大小截断为整数。

**参数：**

- `percents`（默认：`100`）
