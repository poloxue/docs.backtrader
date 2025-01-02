---
title: "参考文档"
weight: 2
---

# Filters 参考文档

## SessionFilter

**class backtrader.filters.SessionFilter(data)**  

此类可作为过滤器应用于数据源，将过滤掉落在常规交易时间之外的日内数据（即盘前/盘后数据）。

- 这是一个“非简单”过滤器，必须管理数据栈（在初始化和调用期间传递）。
- 它不需要“last”方法，因为没有需要传递的内容。

## SessionFilterSimple

**class backtrader.filters.SessionFilterSimple(data)**  

此类可作为过滤器应用于数据源，将过滤掉落在常规交易时间之外的日内数据（即盘前/盘后数据）。

- 这是一个“简单”过滤器，不需要管理数据栈（在初始化和调用期间传递）。
- 它不需要“last”方法，因为没有需要传递的内容。
- Bar 管理将由 SimpleFilterWrapper 类处理，该类在 DataBase.addfilter_simple 调用期间添加。

## SessionFiller

**class backtrader.filters.SessionFiller(data)**  
为声明的会话开始/结束时间内的数据源填充条。

参数：
- **fill_price (默认: None):** 如果传递了 None，将使用前一个条的收盘价。为了得到一个不显示在图表中的条，可以使用 float('NaN')。
- **fill_vol (默认: float('NaN')):** 用于填充缺失交易量的值。
- **fill_oi (默认: float('NaN')):** 用于填充缺失未平仓合约的值。
- **skip_first_fill (默认: True):** 在看到第一个有效条时，不从会话开始填充到该条。

## CalendarDays

**class backtrader.filters.CalendarDays(data)**  

填充缺失日历日到交易日。

参数：

- **fill_price (默认: None):**
  - 0: 用给定值填充。
  - None: 使用上一个已知收盘价。
  - -1: 使用上一个条的中点（高低平均值）。
- **fill_vol (默认: float('NaN')):** 用于填充缺失交易量的值。
- **fill_oi (默认: float('NaN')):** 用于填充缺失未平仓合约的值。

## BarReplayer_Open

**class backtrader.filters.BarReplayer_Open(data)**  

此过滤器将一个条分为两部分：

- **Open:** 条的开盘价将用于交付一个初始价格条，其中四个组件（OHLC）相等。
  - 此初始条的交易量/未平仓合约字段为 0。
- **OHLC:** 原始条完整交付，包含原始交易量/未平仓合约。

分割模拟重播，无需使用重播过滤器。

## DaySplitter_Close

**class backtrader.filters.DaySplitter_Close(data)**  

将一个每日条分为两部分，模拟两个价格点以重播数据：
- **第一个价格点:** OHLX
  - 收盘价将替换为开盘价、高价和低价的平均值。
  - 此价格点使用会话的开盘时间。
- **第二个价格点:** CCCC
  - 收盘价将用于四个价格组件。
  - 此价格点使用会话的收盘时间。
  - 交易量将在两个价格点之间分配，使用参数：
    - **closevol (默认: 0.5):** 值表示百分比，绝对值范围为 0.0 到 1.0，分配给收盘价格点的比例。其余部分将分配给 OHLX 价格点。

此过滤器用于配合 cerebro.replaydata 一起使用。

## HeikinAshi

**class backtrader.filters.HeikinAshi(data)**  
此过滤器重新建模开盘、高价、低价、收盘价以形成 HeikinAshi 蜡烛图。

参考：
- [Heikin Ashi Candlesticks](https://en.wikipedia.org/wiki/Candlestick_chart#Heikin_Ashi_candlesticks)
- [StockCharts Heikin Ashi](http://stockcharts.com/school/doku.php?id=chart_school:chart_analysis:heikin_ashi)

## Renko

**class backtrader.filters.Renko(data)**  

修改数据流以绘制 Renko 条（或砖）。

参数：

- **hilo (默认: False):** 使用高价和低价而不是收盘价来决定是否需要新砖。
- **size (默认: None):** 每个砖块的大小。
- **autosize (默认: 20.0):** 如果 size 为 None，将使用此值自动计算砖块大小（简单地将当前价格除以给定值）。
- **dynamic (默认: False):** 如果为 True 并且使用 autosize，当移动到新砖块时将重新计算砖块大小。这当然会消除 Renko 砖块的完美对齐。
- **align (默认: 1.0):** 用于对齐砖块价格边界的因子。例如，如果价格为 3563.25 并且 align 为 10.0，则对齐后的价格将为 3560。计算方法：
  - 3563.25 / 10.0 = 356.325
  - 四舍五入并删除小数位 -> 356
  - 356 * 10.0 -> 3560

## 参考

- [StockCharts Renko](http://stockcharts.com/school/doku.php?id=chart_school:chart_analysis:renko)
