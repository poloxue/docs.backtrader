---
title: "Interactive Brokers"
weight: 1
---

# 盈透（Interactive Brokers）

与盈透（Interactive Brokers）的集成支持以下功能：

- 实时数据馈送
- 实时交易

**注意**：虽然已尽力测试了各种错误条件，但代码（像任何软件一样）仍可能包含错误。上线前，请使用模拟交易账户或 TWS 演示账户充分测试策略。

**注意**：与盈透的交互通过 IbPy 模块进行，使用前必须先安装。目前 PyPI 中暂没有该模块的包，但可以通过 pip 安装：

```bash
pip install git+https://github.com/blampe/IbPy.git
```

如果你的系统中没有 git（例如在 Windows 上），也可以直接运行：

```bash
pip install https://github.com/blampe/IbPy/archive/master.zip
```

## 示例代码

源码包含一个完整的示例，位于：

`samples/ibtest/ibtest.py`

该示例无法覆盖所有场景，但力求提供广泛的参考，并说明回测模块与实时数据模块在使用上没有本质区别。

注意：

示例在交易活动开始前会等待 `data.LIVE` 状态通知。实现实时策略时通常需要考虑这一点。

## 存储模型与直接模型

与互动经纪商的交互支持两种模型：

- 存储模型（推荐）
- 直接与数据馈送类和经纪商类交互

存储模型清晰分离了经纪商和数据的创建方式。以下两个代码片段展示了具体用法。

首先是存储模型：

```python
import backtrader as bt

ibstore = bt.stores.IBStore(host='127.0.0.1', port=7496, clientId=35)
data = ibstore.getdata(dataname='EUR.USD-CASH-IDEALPRO')
```

参数说明：

- `host`、`port` 和 `clientId` 传入 IBStore 以打开连接。
- `getdata` 创建数据馈送，并通过参数 `dataname` 请求 EUR/USD 外汇对。

直接使用模型：

```python
import backtrader as bt

data = bt.feeds.IBData(dataname='EUR.USD-CASH-IDEALPRO',
                       host='127.0.0.1', port=7496, clientId=35)
```

在这里：

- 参数直接传递给数据，会在后台自动创建 IBStore 实例。

缺点是：

- 不够清晰，因为无法区分哪些参数属于数据、哪些属于存储。

## IBStore - 存储

存储是实时数据馈送/交易支持的核心，提供了 IbPy 与数据馈送、经纪代理之间的适配层。

存储的概念涵盖以下功能：

- 作为中央实体：此处实体为 IB，可以接受或不接受参数。
- 提供获取经纪实例的方法：

```python
IBStore.getbroker(*args, **kwargs)
```

- 提供获取数据馈送实例的方法：

```python
IBStore.getdata(*args, **kwargs)
```

许多 **kwargs 是数据馈送的通用参数，如 `dataname`、`fromdate`、`todate`、`sessionstart`、`sessionend`、`timeframe`、`compression`。

数据可能会提供其他参数。请检查下面的参考。

IBStore 提供：

- 连接目标（`host` 和 `port` 参数）
- 身份识别（`clientId` 参数）
- 重新连接控制（`reconnect` 和 `timeout` 参数）
- 时间偏移检查（`timeoffset` 参数，见下文）
- 通知与调试

```python
notifyall (default: False): IB 发送的任何错误消息（许多仅为信息性）将转发到 Cerebro/Strategy。
_debug (default: False): TWS 接收到的每条消息均打印到标准输出。
```

## IBData 数据馈送

### 数据选项

无论是直接还是通过 `getdata`，IBData 数据馈送都支持以下数据选项：

- 历史下载请求

  如果持续时间超过 IB 为给定时间框架/压缩组合设置的限制，这些请求将被拆分为多个请求。

- 实时数据有三种模式

  - tickPrice 事件（通过 IB `reqMktData`）

    用于 CASH 产品（根据至少 TWS API 9.70 的实验，其他类型不支持）

    通过查看 BID 价格接收 tick 价格事件，根据非官方互联网文献，这似乎是跟踪 CASH 市场价格的方式。

    时间戳在系统中本地生成。如果最终用户希望，可以使用与 IB 服务器时间的偏移（从 IB `reqCurrentTime` 计算）。

  - tickString 事件（即 RTVolume（通过 IB `reqMktData`）

    从 IB 大约每 250 毫秒接收一次 OHLC/成交量快照（如果没有交易发生，则更长时间）。

  - RealTimeBars 事件（通过 IB `reqRealTimeBars`）

    每 5 秒接收一次历史 5 秒的条形图（由 IB 固定的持续时间）。

    如果选择的时间框架/组合低于 Seconds/5 级别，此功能将自动禁用。

    **注意**：`RealTimeBars` 不适用于 TWS 演示。

    默认行为是在大多数情况下使用：`tickString`，除非用户特别希望使用 `RealTimeBars`。

- 回填

  除非用户只请求历史下载，否则数据馈送会自动回填：

  - 启动时：回填最大可能的数据量。例如，对于 Days/1 组合，IB 默认最大持续时间为 1 年，这就是回填的量。

  - 断开连接后：根据断开前收到的最近数据，尽量下载最少的数据来完成回填。

**注意**：最终的时间框架/压缩组合可能不是在创建数据馈送时指定的，而是在插入系统时指定的。请看以下示例：

```python
data = ibstore.getdata(dataname='EUR.USD-CASH-IDEALPRO',
                       timeframe=bt.TimeFrame.Seconds, compression=5)

cerebro.resampledata(data, timeframe=bt.TimeFrame.Minutes, compression=2)
```

现在应该清楚，最终考虑的时间框架/压缩组合是 Minutes/2。

## 数据合约检查

在启动阶段，数据馈送会尝试下载指定合约的详细信息（参阅参考了解如何指定）。如果未找到合约或找到多个匹配项，数据将拒绝继续并通知系统。示例：

```python
data = ibstore.getdata(dataname='TWTR')  # 推特
```

由于默认类型为 `STK`，交易所为 `SMART`，货币为默认（空），将找到一个以 USD 交易的合约（2016-06）。

同样方式对 AAPL 会失败：

```python
data = ibstore.getdata(dataname='AAPL')  # 错误 -> 多个合约
```

因为 `SMART` 找到在多个实际交易所中的合约，且 AAPL 在其中一些以不同货币交易。以下是可以的：

```python
data = ibstore.getdata(dataname='AAPL-STK-SMART-USD')  # 找到1个合约
```

## 数据通知

数据馈送将通过以下方式报告当前状态（查阅 Cerebro 和 Strategy 参考）：

- Cerebro.notify_data（如果覆盖）
- 使用 Cerebro.adddatacb 添加的回调
- Strategy.notify_data（如果覆盖）

策略中的示例如下：

```python
class IBStrategy(bt.Strategy):

    def notify_data(self, data, status, *args, **kwargs):
        if status == data.LIVE:  # 数据已切换为实时数据
           # 执行某些操作
           pass
```

系统状态变化时，会发送以下通知：

- `CONNECTED`：成功初始连接后发送
- `DISCONNECTED`：无法检索数据，系统无法继续。可能的原因：
  - 指定的合约错误
  - 历史下载中断
  - 超过与 TWS 的重连次数
- `CONNBROKEN`：与 TWS 或数据服务器的连接丢失。数据馈送会尝试重新连接，必要时回填并恢复操作。
- `NOTSUBSCRIBED`：合约和连接正常，但因权限不足无法检索数据。系统无法继续。
- `DELAYED`：历史/回填操作进行中，策略处理的数据不是实时数据。
- `LIVE`：策略从此刻开始处理的是实时数据。

策略开发者应考虑处理断连和延迟数据的策略。

## 数据时间框架和压缩

backtrader 生态中的数据馈送在创建时支持 `timeframe` 和 `compression` 参数。这些参数也可作为属性访问，如 `data._timeframe` 和 `data._compression`。

时间框架/压缩组合通过 `resampledata` 或 `replaydata` 传递给 cerebro，告知内部重采样器/重放器目标值。在重采样/重放过程中，`_timeframe` 和 `_compression` 会被覆盖。

但在实时数据馈送中，这些信息可能起关键作用。请看以下示例：

```python
data = ibstore.getdata(dataname='EUR.USD-CASH-IDEALPRO',
                       timeframe=bt.TimeFrame.Ticks,
                       compression=1,  # 1 是默认值
                       rtbar=True,  # 使用实时条
                      )
cerebro.adddata(data)
```

用户请求 tick 数据，这很重要，因为：

- 不会进行回填（IB 支持的最小单位是 Seconds/1）
- 即使请求和支持实时条，如果时间分辨率低于 Seconds/5，也不会使用它们。

除非以 Ticks/1 分辨率工作，否则数据必须重采样/重放。以下是使用实时条的例子：

```python
data = ibstore.getdata(dataname='TWTR-STK-SMART', rtbar=True)
cerebro.resampledata(data, timeframe=bt.TimeFrame.Seconds, compression=20)
```

如上所述，`data._timeframe` 和 `data._compression` 属性将在 `resampledata` 期间被覆盖。以下是将发生的情况：

- 回填将请求 Seconds/20 分辨率
- 实时条将用于实时数据，因为分辨率等于或大于 Seconds/5，数据支持（不是 CASH 产品）
- 来自 TWS 的事件最多每 5 秒发生一次。这可能并不重要，因为系统每 20 秒只会向策略发送一个条。

没有实时条的情况：

```python
data = ibstore.getdata(dataname='TWTR-STK-SMART')
cerebro.resampledata(data, timeframe=bt.TimeFrame.Seconds, compression=20)
```

在这种情况下：

- 回填将请求 Seconds/20 分辨率
- tickString 将用于实时数据（不是 CASH 产品）
- 来自 TWS 的事件最多每 250 毫秒发生一次。这可能并不重要，因为系统每 20 秒只会向策略发送一个条。

最后是 CASH 产品，时间跨度为 20 秒：

```python
data = ibstore.getdata(dataname='EUR.USD-CASH-IDEALPRO')
cerebro.resampledata(data, timeframe=bt.TimeFrame.Seconds, compression=20)
```

在这种情况下：

- 回填将请求 Seconds/20 分辨率
- tickPrice 将用于实时数据，因为这是一个现金产品
- 即使添加了 `rtbar=True`
- 来自 TWS 的事件最多每 250 毫秒发生一次。这可能并不重要，因为系统每 20 秒只会向策略发送一个条。

## 时间管理

数据馈送将自动从 TWS 报告的 ContractDetails 对象中确定时区。

**注意**：这需要安装 `pytz`。如果未安装，用户应提供 `tz` 参数，一个兼容 tzinfo 的实例，用于所需的输出时区。

**注意**：如果安装了 `pytz`，且用户觉得自动时区确定不起作用，`tz` 参数可以包含一个时区名称字符串。backtrader 将尝试使用给定名称实例化一个 `pytz.timezone`。

报告的日期时间将是与产品相关的时区。例如：

- 产品：Eurex 的 EuroStoxxx 50（代码：ESTX50-YYYYMM-DTB）
  - 时区将是 CET（中欧时间），即 Europe/Berlin
- 产品：ES-Mini（代码：ES-YYYYMM-GLOBEX）
  - 时区将是 EST5EDT，即 EST，即 US/Eastern
- 产品：EUR.JPY 外汇对（代码：EUR.JPY-CASH-IDEALPRO）
  - 时区将是 EST5EDT，即 EST，即 US/Eastern

实际上，这是一个互动经纪商的设置，因为外汇对几乎 24 小时不间断交易，因此不会有真实的时区。

这种行为确保无论交易者的实际位置如何，交易都保持一致，因为计算机很可能具有交易地点的实际时区，而不是交易场所的时区。

请阅读手册中的时间管理部分。

**注意**：TWS 演示在报告没有数据下载权限的资产时区方面不准确（EuroStoxx 50 期货是这种情况的一个例子）。

## 实时数据馈送和重采样/重放

实时数据馈送何时交付条？设计原则是：尽可能实时交付。对于 Ticks 时间框架来说确实如此，但如果涉及重采样/重放，可能会有延迟。例如：

重采样配置为 Seconds/5：

```python
cerebro.resampledata(data, timeframe=bt.TimeFrame.Seconds, compression=5)
```

一个时间戳为 23:05:27.325000 的 tick 被传递。

市场交易缓慢，下一个 tick 在 23:05:59.025000 才到来。

问题在于，backtrader 无法预知下一个 tick 要等 32 秒。如果不做处理，时间戳为 23:05:30.000000 的重采样条将延迟约 29 秒才能交付。

因此，实时数据馈送会定期唤醒（每 x 秒），通知重采样器/重放器没有新数据进来。这是由参数 `qcheck`（默认值：0.5 秒）控制的。

这样，重采样器每 `qcheck` 秒就有一次机会交付条。前述场景中的重采样条（23:05:30.000000）最多在报告时间后的 `qcheck` 秒内即可交付。

由于默认值是 0.5 秒，最迟交付时间约为 23:05:30.500000，比原来早了近 29 秒。

缺点是：

某些 tick 可能到达太晚，无法纳入已交付的重采样/重放条。例如，如果在条交付后，TWS 才收到一个时间戳为 23:05:29.995000 的延迟消息，这个消息对已报告的 23:05:30.000000 来说已经太迟。

这主要发生在以下情况：

`timeoffset` 在 IBStore 中被禁用（设为 False）且 IB 报告时间与本地时钟有明显差异时。

避免大多数延迟样本的最佳方法：

增大 `qcheck` 值，给延迟消息留出空间：

```python
data = ibstore.getdata('TWTR', qcheck=2.0, ...)
```

这应该增加额外的空间，即使它会延迟重采样/重放条的交付。

**注意**：当然，对于 Seconds/5 重采样来说，2.0 秒的延迟与 Minutes/10 的重采样有不同的意义。

如果希望禁用 `timeoffset` 并且不通过 `qcheck` 管理，仍然可以处理延迟样本：

使用 `_latethrough` 设置为 True 作为 `getdata / IBData` 的参数：

```python
data = ibstore.getdata('TWTR', _latethrough=True, ...)
```

在重采样/重放时使用 `takelate` 设置为 True：

```python
cerebro.resampledata(data, takelate=True)
```

## IBBroker - 实时交易

**注意**：应需求，模拟经纪商中实现了 `tradeid` 功能，用于正确分配不同 `tradeid` 的佣金。

但实时经纪商无法做到这一点，因为佣金是在无法区分不同 `tradeid` 的时间点报告的。

因此，虽然仍可以指定 `tradeid`，但已不再有意义。

### 使用经纪商

要使用 IB 经纪商，必须替换 cerebro 创建的标准模拟经纪实例。

使用存储模型（推荐）：

```python
import backtrader as bt

cerebro = bt.Cerebro()
ibstore = bt.stores.IBStore(host='127.0.0.1', port=7496, clientId=35)
cerebro.broker = ibstore.getbroker()  # 或者 cerebro.setbroker(...)
```

直接使用方法：

```python
import backtrader as bt

cerebro = bt.Cerebro()
cerebro.broker = bt.brokers.IBBroker(host='127.0.0.1', port=7496, clientId=35)
```

## 经纪商参数

无论是直接还是通过 `getbroker`，IBBroker 都不支持任何参数。因为经纪商只是代理，不应屏蔽真实经纪商提供的功能。

## 一些限制

### 现金和价值报告

内部模拟经纪商在策略 `next` 方法之前计算净值和现金，而实时经纪商无法保证这一点。

如果请求这些值，`next` 的执行可能会延迟到收到响应。

经纪商可能尚未计算这些值。

backtrader 通知 TWS 提供更新值，但无法预知消息何时到达。

`IBBroker` 的 `getcash` 和 `getvalue` 方法始终返回从 IB 收到的最新值。

**注意**：另一个限制是，这些值以账户的基础货币报告，即使有其他货币的值。这是设计上的选择。

### 头寸

backtrader 使用 TWS 报告的资产头寸（价格和数量）。也可以通过订单执行和状态消息内部计算，但如果丢失了部分消息，计算结果将不准确。

此外，如果连接 TWS 时资产已有未平仓头寸，策略计算的交易将因初始偏移而无法正常工作。

## 交易

在使用方面没有变化。只需使用策略中提供的方法（请参阅策略参考以获取完整说明）：

- `buy`
- `sell`
- `close`
- `cancel`

## 返回的订单对象

与 backtrader 订单对象兼容（在相同层次结构中子类化）。

## 订单执行类型

IB 支持多种执行类型，部分由 IB 模拟，部分由交易所支持。选择支持哪些类型的动机是：

与 backtrader 模拟经纪商兼容

这样，上线的内容都经过了回测验证。

因此，订单执行类型限于模拟经纪商中可用的类型：

- `Order.Market`
- `Order.Close`
- `Order.Limit`
- `Order.Stop`（触发止损后跟随一个市价单）
- `Order.StopLimit`（触发止损后跟随一个限价单）

**注意**：IB 根据策略触发止损。backtrader 不修改默认设置（0）：

0 - 默认值。对于 OTC 股票和美国期权，使用”双重买/卖”方法；其他订单使用”最后”方法。
如需修改此设置，可以按照 IB 文档传入额外的 **kwargs。例如，在策略的 `next` 方法中：

```python
def next(self):
    # 一些逻辑
    self.buy(data, m_triggerMethod=2)
```

这将策略改为 2（“最后”方法，其中止损订单基于最后价格触发）。

请参考 IB API 文档以获取关于止损触发的进一步说明。

## 订单有效期

回测中使用的有效期概念（`valid` 参数）在实时交易中同样适用。有效期参数转换为 IB 订单的方式如下：

- `None` -> `GTC`（Good Til Cancelled）

  未指定有效期，表示订单有效直至取消。

- `datetime/date` -> `GTD`（Good Til Date）

  传入 `datetime.datetime` 或 `datetime.date` 实例表示订单有效至指定时间点。

- `timedelta(x)` -> `GTD`（`timedelta(x)` 不等于 `timedelta()`）

  表示订单有效期为当前时间加上 `timedelta(x)`。

- `float` -> `GTD`

  如果值来自 backtrader 的浮点日期时间存储，订单有效至该浮点数指示的时间。

- `timedelta()` 或 0 -> `DAY`

  指定了值（非 `None`）但为空值，表示当日（当前会话）有效的订单。

## 通知

标准订单状态通过策略的 `notify_order` 方法通知（如果覆盖）。

- `Submitted` - 订单已发送到 TWS
- `Accepted` - 订单已被放置
- `Rejected` - 订单放置失败或在其生命周期内被系统取消
- `Partial` - 部分执行已发生
- `Completed` - 订单已完全执行
- `Canceled`（或 `Cancelled`）

  在 IB 中有多种含义：

  - 用户手动取消
  - 服务器/交易所取消订单
  - 订单有效期已过

  会应用启发式方法：如果从 TWS 收到 `openOrder` 消息且状态为 `PendingCancel` 或 `Canceled`，订单将被标记为已过期。

- `Expired` - 参见上文解释。

## 参考

### IBStore

```python
class backtrader.stores.IBStore()
```

包装 ibpy `ibConnection` 实例的单例类。

这些参数也可以在使用该存储的类（如 `IBData` 和 `IBBroker`）中指定。

参数：

- `host`（默认：`127.0.0.1`）：IB TWS 或 IB Gateway 的运行地址。通常为 localhost，但不一定。
- `port`（默认：7496）：连接端口。演示系统使用 7497。
- `clientId`（默认：无）：TWS 连接标识。
  - 无：自动生成 1 到 65535 之间的随机 id
  - 整数：直接使用该值。
- `notifyall`（默认：False）
  - 为 False 时，仅将错误消息发送到 `notify_store` 方法。
  - 为 True 时，转发所有从 TWS 收到的消息。
- `_debug`（默认：False）
  - 将所有收到的 TWS 消息打印到标准输出。
- `reconnect`（默认：3）
  - 初始连接失败后的重试次数。
  - 设为 -1 表示无限重连。
- `timeout`（默认：3.0）
  - 重连间隔（秒）。
- `timeoffset`（默认：True）
  - 为 True 时，使用从 `reqCurrentTime`（IB 服务器时间）计算与本地时间的偏移，并将该偏移用于价格通知的时间戳修正（如 CASH 市场的 `tickPrice` 事件）。
  - 时间偏移会传播到其他模块（如重采样），以对齐时间戳。
- `timerefresh`（默认：60.0）
  - 时间偏移刷新间隔（秒）。
- `indcash`（默认：True）
  - 将 IND 代码作为现金资产处理以获取价格。

## IBBroker

```python
class backtrader.brokers.IBBroker(**kwargs)
```

盈透的经纪商实现。

该类将盈透的订单/头寸映射到 backtrader 的内部 API。

注意事项：
- `tradeid` 实际不受支持，因为盈亏直接从盈透获取，以 FIFO 方式计算，导致 `tradeid` 的盈亏不准确。
- 头寸：如果启动时已有未平仓头寸，或通过其他方式改变了头寸，Cerebro 中的策略交易将不反映实际情况。
  - 若要避免此问题，经纪商需自行管理头寸（这也支持多个 `tradeid` 的盈亏本地计算），但这可能违背使用实时经纪商的初衷。

## IBData

```python
class backtrader.feeds.IBData(**kwargs)
```

盈透数据馈送。

`dataname` 参数支持以下合约规格：

- `TICKER` # 股票类型和 SMART 交易所
- `TICKER-STK` # 股票和 SMART 交易所
- `TICKER-STK-EXCHANGE` # 股票
- `TICKER-STK-EXCHANGE-CURRENCY` # 股票
- `TICKER-CFD` # CFD 和 SMART 交易所
- `TICKER-CFD-EXCHANGE` # CFD
- `TICKER-CDF-EXCHANGE-CURRENCY` # 股票
- `TICKER-IND-EXCHANGE` # 指数
- `TICKER-IND-EXCHANGE-CURRENCY` # 指数
- `TICKER-YYYYMM-EXCHANGE` # 期货
- `TICKER-YYYY

MM-EXCHANGE-CURRENCY` # 期货
- `TICKER-YYYYMM-EXCHANGE-CURRENCY-MULT` # 期货
- `TICKER-FUT-EXCHANGE-CURRENCY-YYYYMM-MULT` # 期货
- `TICKER-YYYYMM-EXCHANGE-CURRENCY-STRIKE-RIGHT` # 期权
- `TICKER-YYYYMM-EXCHANGE-CURRENCY-STRIKE-RIGHT-MULT` # 期权
- `TICKER-FOP-EXCHANGE-CURRENCY-YYYYMM-STRIKE-RIGHT` # 期权
- `TICKER-FOP-EXCHANGE-CURRENCY-YYYYMM-STRIKE-RIGHT-MULT` # 期权
- `CUR1.CUR2-CASH-IDEALPRO` # 外汇
- `TICKER-YYYYMMDD-EXCHANGE-CURRENCY-STRIKE-RIGHT` # 期权
- `TICKER-YYYYMMDD-EXCHANGE-CURRENCY-STRIKE-RIGHT-MULT` # 期权
- `TICKER-OPT-EXCHANGE-CURRENCY-YYYYMMDD-STRIKE-RIGHT` # 期权
- `TICKER-OPT-EXCHANGE-CURRENCY-YYYYMMDD-STRIKE-RIGHT-MULT` # 期权

参数：

- `sectype`（默认：STK）
  - 在 `dataname` 中未指定时应用的默认证券类型。
- `exchange`（默认：SMART）
  - 在 `dataname` 中未指定时应用的默认交易所。
- `currency`（默认：''）
  - 在 `dataname` 中未指定时应用的默认货币。
- `historical`（默认：False）
  - 设为 True 时，数据馈送在首次下载数据后停止。
  - 使用 `fromdate` 和 `todate` 作为参考。
  - 如果请求持续时间超过 IB 在给定时间框架/压缩下的限制，将自动分多次请求。
- `what`（默认：None）
  - 为 None 时，历史数据请求的默认值因资产类型而异：
    - CASH 资产：'BID'
    - 其他：'TRADES'
  - 如需其他值，请查阅 IB API 文档。
- `rtbar`（默认：False）
  - 为 True 时，使用盈透提供的 5 秒实时条作为最小刻度。这些条形图对应经过 IB 处理后的实时值。
  - 为 False 时，使用基于接收 tick 的 RTVolume 价格。CASH 资产（如 EUR.JPY）始终使用 RTVolume，从中提取买入价格（这已成为与 IB 协作的行业默认做法）。
  - 即使设为 True，如果数据重采样到低于 Seconds/5 的时间框架，也不会使用实时条，因为 IB 不提供更细粒度。
- `qcheck`（默认：0.5）
  - 无数据接收时的唤醒间隔（秒），用于重采样/重放和通知传递。
- `backfill_start`（默认：True）
  - 启动时执行回填。通过单个请求获取最大历史数据量。
- `backfill`（默认：True）
  - 断开/重连后执行回填。按缺口时长下载最小数据量。
- `backfill_from`（默认：None）
  - 可用于初始回填的备用数据源。数据源耗尽后，如有需要再从 IB 回填。典型用法是使用本地文件等持久化来源。
- `latethrough`（默认：False）
  - 数据源被重采样/重放时，部分 tick 可能迟到。设为 True 允许这些延迟 tick 通过。
  - 请查阅重采样器文档了解如何处理这些 tick。
  - 此情况在 `timeoffset` 设为 False 且 TWS 服务器时间与本地时间不同步时可能发生。
- `tradename`（默认：None）
  - 某些场景下（如 CFD），价格由一个资产提供，交易却在另一个资产中进行。

  ```python
  SPY-STK-SMART-USD -> SP500 ETF（将指定为 `dataname`）
  SPY-CFD-SMART-USD -> 对应的 CFD，不提供价格跟踪，但在这种情况下将是交易资产（指定为 `tradename`）。
  ```

默认参数允许 `TICKER` 这样的简写，其中 `sectype`（STK）和 `exchange`（SMART）使用默认值。

部分资产（如 AAPL）需要完整的规格说明（包含货币），而其他资产（如 TWTR）可以直接使用简写。

完整规格示例：`dataname='AAPL-STK-SMART-USD'`

或者：`IBData(dataname='AAPL', currency='USD')` 使用默认的 STK 和 SMART，仅覆盖货币为 USD。
