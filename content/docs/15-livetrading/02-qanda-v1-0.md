---
title: "Qanda"
weight: 2
---

# Oanda
Oanda的集成支持以下功能：

- 实时数据馈送
- 实时交易

## 要求

- **oandapy**：可以通过以下命令安装：`pip install git+https://github.com/oanda/oandapy.git`
- **pytz**（可选，不推荐）：由于外汇市场的全球性和24x7的特点，选择使用UTC时间。如果愿意，仍然可以使用所需的输出时区。

## 示例代码

源代码中包含完整示例：

`samples/oandatest/oandatest.py`

## Oanda - 存储

存储是实时数据馈送和交易支持的核心，提供了Oanda API和数据馈送及经纪代理之间的适配层。

可以通过以下方法获取经纪商实例：
```python
OandaStore.getbroker(*args, **kwargs)
```

可以通过以下方法获取数据馈送实例：
```python
OandaStore.getdata(*args, **kwargs)
```
在这种情况下，许多**kwargs是数据馈送的常见参数，如dataname、fromdate、todate、sessionstart、sessionend、timeframe、compression。

数据可能提供其他参数。请参阅下面的参考。

## 必要参数

为了成功连接到Oanda，以下参数是必要的：

- `token`（默认：无）：API访问令牌
- `account`（默认：无）：账户ID

这些由Oanda提供。

无论是连接到模拟服务器还是真实服务器，请使用：
- `practice`（默认：False）：使用测试环境

账户需要定期检查以获取现金和价值。周期性可以通过以下方式控制：
- `account_tmout`（默认：10.0）：刷新账户价值/现金刷新周期

## Oanda数据馈送

实例化数据时：

按照Oanda指南传递符号。例如，EUR/USD根据Oanda的指南需要指定为EUR_USD。实例化如下：
```python
data = oandastore.getdata(dataname='EUR_USD', ...)
```

## 时间管理

除非传递了`tz`参数（一个pytz兼容对象）给数据馈送，否则所有时间输出都以UTC格式表示，如上所述。

## 回填
backtrader不会向Oanda发出特殊请求。对于小时间框架，Oanda在模拟服务器上返回的回填数据长度为500个柱。

## OandaBroker - 实时交易

### 使用经纪商

要使用OandaBroker，需要替换cerebro创建的标准经纪商模拟实例。

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
使用方面没有变化。只需使用策略中提供的方法（请参阅策略参考以获取完整说明）：
- `buy`
- `sell`
- `close`
- `cancel`

### 订单执行类型
Oanda支持几乎所有backtrader需要的订单执行类型，除了Close。因此，订单执行类型限于：
- `Order.Market`
- `Order.Limit`
- `Order.Stop`
- `Order.StopLimit`（使用Stop和upperBound/lowerBound价格）
- `Order.StopTrail`

通过使用`takeprofit`和`stoploss`订单成员并创建内部模拟订单来支持括号订单。

### 订单有效期
backtrader在回测期间可用的相同有效期概念（使用`valid`参数买入和卖出）在此也可用，并具有相同含义。因此，Oanda订单的有效期参数转换如下：
- `None` 转换为 `Good Til Cancelled`
  因为未指定有效期，理解为订单必须有效直到取消。
- `datetime/date` 转换为 `Good Til Date`
- `timedelta(x)` 转换为 `Good Til Date`（这里`timedelta(x)`不等于`timedelta()`）
  解释为订单有效从现在起加上`timedelta(x)`。
- `timedelta()` 或 0 转换为 `Session`
  传递了一个值（而不是`None`），但为空值，解释为当天（会话）有效的订单。

### 通知
标准订单状态将通过策略的`notify_order`方法通知（如果覆盖）。
- `Submitted` - 订单已发送到TWS
- `Accepted` - 订单已被放置
- `Rejected` - 用于真正的拒绝以及在订单创建过程中没有其他已知状态时
- `Partial` - 部分执行已发生
- `Completed` - 订单已完全执行
- `Canceled`（或 `Cancelled`）
- `Expired` - 当订单因过期而取消时

## 参考

### OandaStore
```python
class backtrader.stores.OandaStore()
```
控制与Oanda连接的单例类。

参数：
- `token`（默认：无）：API访问令牌
- `account`（默认：无）：账户ID
- `practice`（默认：False）：使用测试环境
- `account_tmout`（默认：10.0）：刷新账户价值/现金刷新周期

### OandaBroker
```python
class backtrader.brokers.OandaBroker(**kwargs)
```
Oanda的经纪商实现。

该类将Oanda的订单/头寸映射到backtrader的内部API。

参数：
- `use_positions`（默认：True）：连接到经纪商提供商时使用现有头寸来启动经纪商。
  在实例化时设置为False以忽略任何现有头寸。

### OandaData

```python
class backtrader.feeds.OandaData(**kwargs)
```
Oanda数据馈送。

参数：
- `qcheck`（默认：0.5）：在没有数据接收时唤醒的时间（秒），以便适当地重采样/重放数据包并将通知上传到链中。
- `historical`（默认：False）：如果设置为True，数据馈送将在首次下载数据后停止。
  将使用标准数据馈送参数`fromdate`和`todate`作为参考。
  如果请求的持续时间大于IB在给定时间框架/压缩组合下允许的持续时间，数据馈送将进行多次请求。
- `backfill_start`（默认：True）：在启动时执行回填。将通过单个请求获取最大可能的历史数据。
- `backfill`（默认：True）：在断开/重新连接周期后执行回填。将使用间隙持续时间下载最小可能的数据量。
- `backfill_from`（默认：None）：可以传递另一个数据源以进行初始回填。一旦数据源耗尽，如果需要，将从IB回填。理想情况下，这意味着从已存储的来源（如磁盘上的文件）进行回填，但不限于此。
- `bidask`（默认：True）：如果为True，则历史/回填请求将请求服务器的买卖价。
  如果为False，则请求中间价。
- `useask`（默认：False）：如果为True，则使用买卖价的卖价部分，而不是默认使用买价。
- `includeFirst`（默认：True）：通过直接将参数设置为Oanda API调用，影响历史/回填请求的第一个柱的交付。
- `reconnect`（默认：True）：当网络连接断开时重新连接。
- `reconnections`（默认：-1）：尝试重新连接的次数：-1表示永远。
- `reconntimeout`（默认：5.0）：重新连接尝试之间的等待时间（秒）。

此数据馈送仅支持以下时间框架和压缩的映射，这符合OANDA API开发者指南中的定义：

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
