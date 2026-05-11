---
title: "Qanda"
weight: 2
---

# Oanda

Oanda 的集成支持以下功能：

- 实时数据馈送
- 实时交易

## 要求

- **oandapy**：安装命令：`pip install git+https://github.com/oanda/oandapy.git`
- **pytz**（可选，不推荐）：由于外汇市场 24x7 全球交易，默认使用 UTC 时间。如有需要，仍可指定输出时区。

## 示例代码

源代码中包含完整示例：

`samples/oandatest/oandatest.py`

## Oanda - 存储

存储是实时数据馈送和交易支持的核心，提供了 Oanda API 与数据馈送、经纪代理之间的适配层。

可以通过以下方法获取经纪商实例：
```python
OandaStore.getbroker(*args, **kwargs)
```

可以通过以下方法获取数据馈送实例：
```python
OandaStore.getdata(*args, **kwargs)
```
许多 **kwargs 是数据馈送的通用参数，如 `dataname`、`fromdate`、`todate`、`sessionstart`、`sessionend`、`timeframe`、`compression`。

数据可能提供其他参数。请参阅下面的参考。

## 必要参数

连接到 Oanda 需要以下参数：

- `token`（默认：无）：API 访问令牌
- `account`（默认：无）：账户 ID

这些由 Oanda 提供。

连接模拟或真实服务器：
- `practice`（默认：False）：使用测试环境

账户需要定期检查现金和价值，可通过以下参数控制刷新频率：
- `account_tmout`（默认：10.0）：账户价值/现金刷新周期（秒）

## Oanda 数据馈送

实例化数据时，按照 Oanda 指南传递符号。例如，EUR/USD 需要指定为 `EUR_USD`：
```python
data = oandastore.getdata(dataname='EUR_USD', ...)
```

## 时间管理

除非向数据馈送传递了 `tz` 参数（pytz 兼容对象），否则所有时间输出均为 UTC 格式。

## 回填

backtrader 不会向 Oanda 发出特殊请求。小时间框架下，模拟服务器返回的回填数据长度为 500 根 K 线。

## OandaBroker - 实时交易

### 使用经纪商

要使用 OandaBroker，需替换 cerebro 创建的默认模拟经纪商。

使用存储模型（推荐）：
```python
import backtrader as bt

cerebro = bt.Cerebro()
oandastore = bt.stores.OandaStore()
cerebro.broker = oandastore.getbroker()  # 或 cerebro.setbroker(...)
```

### 经纪商 - 初始头寸

经纪商支持一个参数：
- `use_positions`（默认：True）：连接到经纪商提供商时使用现有头寸来启动经纪商。
  在实例化时设置为False以忽略任何现有头寸。

### 操作
使用方法与回测一致。使用策略中的标准方法：
- `buy`
- `sell`
- `close`
- `cancel`

### 订单执行类型
Oanda 支持 backtrader 所需的大部分订单执行类型（Close 除外）。支持的订单类型：
- `Order.Market`
- `Order.Limit`
- `Order.Stop`
- `Order.StopLimit`（使用 Stop 和 upperBound/lowerBound 价格）
- `Order.StopTrail`

通过 `takeprofit` 和 `stoploss` 订单成员创建内部模拟订单来支持括号订单。

### 订单有效期
回测中的有效期概念（`valid` 参数）在此同样适用。Oanda 订单的有效期转换如下：
- `None` -> `Good Til Cancelled`
  未指定有效期，订单有效直至取消。
- `datetime/date` -> `Good Til Date`
- `timedelta(x)` -> `Good Til Date`（`timedelta(x)` 不等于 `timedelta()`）
  表示订单有效期为当前时间加上 `timedelta(x)`。
- `timedelta()` 或 0 -> `Session`
  已指定值（非 `None`）但为空值，表示当日（当前会话）有效。

### 通知
标准订单状态通过策略的 `notify_order` 方法通知（如果覆盖）。
- `Submitted` - 订单已发送
- `Accepted` - 订单已放置
- `Rejected` - 表示实际拒绝或在创建过程中无其他状态时使用
- `Partial` - 已部分执行
- `Completed` - 订单已完全执行
- `Canceled`（或 `Cancelled`）
- `Expired` - 订单因过期取消

## 参考

### OandaStore

```python
class backtrader.stores.OandaStore()
```

控制与 Oanda 连接的单例类。

参数：
- `token`（默认：无）：API 访问令牌
- `account`（默认：无）：账户 ID
- `practice`（默认：False）：使用测试环境
- `account_tmout`（默认：10.0）：账户价值/现金刷新周期（秒）

### OandaBroker

```python
class backtrader.brokers.OandaBroker(**kwargs)
```

Oanda 的经纪商实现，将 Oanda 的订单/头寸映射到 backtrader 的内部 API。

参数：
- `use_positions`（默认：True）：连接时使用现有头寸初始化经纪商。
  设为 False 可忽略任何现有头寸。

### OandaData

```python
class backtrader.feeds.OandaData(**kwargs)
```

Oanda 数据馈送。

参数：
- `qcheck`（默认：0.5）：无数据时的唤醒间隔（秒），用于重采样/重放和通知传递。
- `historical`（默认：False）：设为 True 时，首次下载数据后停止。
  使用 `fromdate` 和 `todate` 作为参考。
  如果请求时长超过 Oanda 在给定时间框架/压缩下的限制，将自动分段请求。
- `backfill_start`（默认：True）：启动时执行回填，通过单次请求获取最大历史数据。
- `backfill`（默认：True）：断开/重连后执行回填，按缺口时长下载最小数据量。
- `backfill_from`（默认：None）：可用于初始回填的备用数据源。数据源耗尽后，如有需要再从 Oanda 回填。
- `bidask`（默认：True）：历史/回填请求返回买卖价。设为 False 则返回中间价。
- `useask`（默认：False）：使用卖价而非默认的买价。
- `includeFirst`（默认：True）：控制历史/回填请求中第一个 K 线的返回行为。
- `reconnect`（默认：True）：网络断开时自动重连。
- `reconnections`（默认：-1）：重连次数，-1 表示无限重试。
- `reconntimeout`（默认：5.0）：重连间隔（秒）。

此数据馈送支持的时间框架和压缩映射如下（符合 OANDA API 开发者指南）：

- `(TimeFrame.Seconds, 5): 'S5'`
- `(TimeFrame.Seconds, 10): 'S10'`
- `(TimeFrame.Seconds, 15): 'S15'`
- `(TimeFrame.Seconds, 30): 'S30'`
- `(TimeFrame.Minutes, 1): 'M1'`
- `(TimeFrame.Minutes, 2): 'M3'`
- `(TimeFrame.Minutes, 3): 'M3'`
- `(TimeFrame.Minutes, 4): 'M4'`
- `(TimeFrame.Minutes, 5): 'M5'`
- `(TimeFrame.Minutes, 10): 'M10'`
- `(TimeFrame.Minutes, 15): 'M15'`
- `(TimeFrame.Minutes, 30): 'M30'`
- `(TimeFrame.Minutes, 60): 'H1'`
- `(TimeFrame.Minutes, 120): 'H2'`
- `(TimeFrame.Minutes, 180): 'H3'`
- `(TimeFrame.Minutes, 240): 'H4'`
-

 `(TimeFrame.Minutes, 360): 'H6'`
- `(TimeFrame.Minutes, 480): 'H8'`
- `(TimeFrame.Days, 1): 'D'`
- `(TimeFrame.Weeks, 1): 'W'`
- `(TimeFrame.Months, 1): 'M'`

其他任何组合将被拒绝。
