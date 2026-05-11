---
title: "参考文档"
weight: 2
---

# Filters 参考文档

## SessionFilter

**class backtrader.filters.SessionFilter(data)**  

过滤掉常规交易时间外的日内数据（即盘前/盘后数据）。

- 这是一个”非简单”过滤器，必须管理数据栈。
- 不需要 “last” 方法，因为没有需要推送的数据。

## SessionFilterSimple

**class backtrader.filters.SessionFilterSimple(data)**  

过滤掉常规交易时间外的日内数据（即盘前/盘后数据）。

- 这是一个”简单”过滤器，不需要管理数据栈。
- 不需要 “last” 方法。
- Bar 管理由 `SimpleFilterWrapper` 类处理，在 `DataBase.addfilter_simple` 调用期间添加。

## SessionFiller

**class backtrader.filters.SessionFiller(data)**  
为声明的会话开始/结束时间内的数据源填充 Bar。

参数：
- **fill_price (默认: None):** 为 None 时使用前一个 Bar 的收盘价。使用 `float('NaN')` 可生成不显示在图表中的 Bar。
- **fill_vol (默认: float('NaN')):** 用于填充缺失交易量的值。
- **fill_oi (默认: float('NaN')):** 用于填充缺失未平仓合约的值。
- **skip_first_fill (默认: True):** 看到第一个有效 Bar 时，不从会话开始填充到此 Bar。

## CalendarDays

**class backtrader.filters.CalendarDays(data)**  

填充缺失的日历日到交易日。

参数：

- **fill_price (默认: None):**
  - 0: 用给定值填充。
  - None: 使用上一个已知收盘价。
  - -1: 使用上一个 Bar 的中点（高/低平均值）。
- **fill_vol (默认: float('NaN')):** 用于填充缺失交易量的值。
- **fill_oi (默认: float('NaN')):** 用于填充缺失未平仓合约的值。

## BarReplayer_Open

**class backtrader.filters.BarReplayer_Open(data)**  

将一个 Bar 分为两部分：

- **Open:** 开盘价用于交付一个初始价格条，四个组件（OHLC）相等。
  - 初始条的交易量/未平仓合约为 0。
- **OHLC:** 原始条完整交付，包含原始交易量/未平仓合约。

分割模拟重播，无需使用重播过滤器。

## DaySplitter_Close

**class backtrader.filters.DaySplitter_Close(data)**  

将一个每日 Bar 分为两部分，模拟两个价格点以重播数据：
- **第一个价格点:** OHLX
  - 收盘价替换为开盘价、最高价、最低价的平均值。
  - 使用会话的开盘时间。
- **第二个价格点:** CCCC
  - 收盘价用于四个价格组件。
  - 使用会话的收盘时间。
  - 交易量在两个价格点之间分配：
    - **closevol (默认: 0.5):** 分配给收盘价格点的比例（0.0 到 1.0），其余分配给 OHLX 价格点。

此过滤器与 `cerebro.replaydata` 配合使用。

## HeikinAshi

**class backtrader.filters.HeikinAshi(data)**  
重新建模开盘价、最高价、最低价、收盘价以形成 HeikinAshi 蜡烛图。

参考：
- [Heikin Ashi Candlesticks](https://en.wikipedia.org/wiki/Candlestick_chart#Heikin_Ashi_candlesticks)
- [StockCharts Heikin Ashi](http://stockcharts.com/school/doku.php?id=chart_school:chart_analysis:heikin_ashi)

## Renko

**class backtrader.filters.Renko(data)**  

修改数据流以绘制 Renko 砖。

参数：

- **hilo (默认: False):** 使用最高价和最低价代替收盘价来决定是否需要新砖。
- **size (默认: None):** 每个砖块的大小。
- **autosize (默认: 20.0):** 如果 size 为 None，用此值自动计算砖块大小（当前价格除以该值）。
- **dynamic (默认: False):** 如果为 True 且使用 autosize，移动到新砖块时重新计算砖块大小。这会破坏 Renko 砖的完美对齐。
- **align (默认: 1.0):** 砖块价格边界的对齐因子。例如，价格为 3563.25、align 为 10.0 时，对齐后的价格为 3560：
  - 3563.25 / 10.0 = 356.325
  - 四舍五入取整 -> 356
  - 356 * 10.0 -> 3560

## 参考

- [StockCharts Renko](http://stockcharts.com/school/doku.php?id=chart_school:chart_analysis:renko)
