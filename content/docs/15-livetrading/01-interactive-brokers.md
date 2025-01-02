---
title: "Interactive Brokers"
weight: 1
---

### 盈透（Interactive Brokers）

与盈透（Interactive Brokers）的集成支持以下功能：

- 实时数据馈送
- 实时交易

**注意**：尽管已经尽力测试了尽可能多的错误条件和情况，但代码（像任何其他软件一样）可能包含错误。在进入生产环境之前，请使用纸面交易账户或 TWS 演示帐户彻底测试任何策略。

**注意**：与互动经纪商的交互是通过使用 IbPy 模块进行的，该模块在使用前必须安装。目前在 Pypi 中没有该模块的包（撰写本文时），但可以使用以下命令通过 pip 安装：

```bash
pip install git+https://github.com/blampe/IbPy.git
```

如果您的系统中没有 git（例如在 Windows 上安装），以下命令也可以正常工作：

```bash
pip install https://github.com/blampe/IbPy/archive/master.zip
```

### 示例代码

源码包含一个完整的示例，位于：

`samples/ibtest/ibtest.py`

该示例无法涵盖所有可能的用例，但它试图提供广泛的见解，并应强调在使用回测模块或实时数据模块时没有实际差异。

需要注意的一点是：

示例在任何交易活动开始之前，都会等待 `data.LIVE` 数据状态通知。这可能是任何实时策略中都需要考虑的事项。

### 存储模型与直接模型

与互动经纪商的交互支持两种模型：

- 存储模型（推荐）
- 直接与数据馈送类和经纪商类交互

存储模型提供了一种明确的分离模式，用于创建经纪商和数据。以下两个代码片段应更好地作为示例。

首先是存储模型：

```python
import backtrader as bt

ibstore = bt.stores.IBStore(host='127.0.0.1', port=7496, clientId=35)
data = ibstore.getdata(dataname='EUR.USD-CASH-IDEALPRO')
```

这里的参数：

- `host`，`port` 和 `clientId` 传递到 IBStore 中，用于打开连接。
- 然后使用 `getdata` 创建数据馈送，并使用 backtrader 中所有数据馈送中常见的参数 `dataname` 请求 EUR/USD 外汇对。

直接使用模型：

```python
import backtrader as bt

data = bt.feeds.IBData(dataname='EUR.USD-CASH-IDEALPRO',
                       host='127.0.0.1', port=7496, clientId=35)
```

在这里：

- 参数直接传递给数据。这些将用于在后台创建 IBStore 实例。

缺点是：

- 清晰度较低，因为不清楚哪些属于数据，哪些属于存储。

### IBStore - 存储

存储是实时数据馈送/交易支持的关键，提供了 IbPy 模块和数据馈送及经纪代理需求之间的适配层。

存储是一个涵盖以下功能的概念：

- 作为一个实体的中央商店：在这种情况下，实体是 IB，可以需要或不需要参数。
- 提供访问获取经纪实例的方法：

```python
IBStore.getbroker(*args, **kwargs)
```

- 提供访问获取数据馈送实例的方法：

```python
IBStore.getdata(*args, **kwargs)
```

在这种情况下，许多 **kwargs 是数据馈送中常见的，如 `dataname`，`fromdate`，`todate`，`sessionstart`，`sessionend`，`timeframe`，`compression`。

数据可能会提供其他参数。请检查下面的参考。

IBStore 提供：

- 连接目标（`host` 和 `port` 参数）
- 身份识别（`clientId` 参数）
- 重新连接控制（`reconnect` 和 `timeout` 参数）
- 时间偏移检查（`timeoffset` 参数，见下文）
- 通知和调试

```python
notifyall (default: False): 在这种情况下，IB 发送的任何错误信息（许多只是信息性）将被转发到 Cerebro/Strategy。
_debug (default: False): 在这种情况下，TWS 接收到的每条消息都将打印到标准输出。
```

### IBData 数据馈送

#### 数据选项

无论是直接还是通过 `getdata`，IBData 数据馈送都支持以下数据选项：

- 历史下载请求

  如果持续时间超过 IB 为给定时间框架/压缩组合设置的限制，这些请求将被拆分为多个请求。

- 实时数据有三种口味

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

  除非用户请求只进行历史下载，否则数据馈送将自动回填：

  - 启动时：最大可能持续时间。例如，对于 Days/1（时间框架/压缩）组合，IB 的默认最大持续时间为 1 年，这是将被回填的时间量。

  - 数据断开连接后：在这种情况下，通过查看断开连接前接收的最新数据，回填操作下载的数据量将减少到最小。

**注意**：请考虑最终时间框架/压缩组合可能不是在创建数据馈送时指定的，而是在系统中插入时指定的。请看以下示例：

```python
data = ibstore.getdata(dataname='EUR.USD-CASH-IDEALPRO',
                       timeframe=bt.TimeFrame.Seconds, compression=5)

cerebro.resampledata(data, timeframe=bt.TimeFrame.Minutes, compression=2)
```

现在应该清楚，最终考虑的时间框架/压缩组合是 Minutes/2。

### 数据合约检查

在启动阶段，数据馈送将尝试下载指定合约的详细信息（请参阅参考，了解如何指定）。如果未找到合约或找到多个匹配项，数据将拒绝继续并将通知系统。一些示例。

简单但明确的合约规格：

```python
data = ibstore.getdata(dataname='TWTR')  # 推特
```

由于默认类型 `STK`，交易所 `SMART` 和货币（默认无），将找到一个在 USD 中交易的单一合约（2016-06）。

类似的方式对 AAPL 会失败：

```python
data = ibstore.getdata(dataname='AAPL')  # 错误 -> 多个合约
```

因为 `SMART` 找到在多个实际交易所中的合约，且 AAPL 在其中一些以不同货币交易。以下是可以的：

```python
data = ibstore.getdata(dataname='AAPL-STK-SMART-USD')  # 找到1个合约
```

### 数据通知

数据馈送将通过以下一种或多种方式报告当前状态（请检查 Cerebro 和 Strategy 参考）：

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

在系统中发生变化时，将发送以下通知：

- `CONNECTED`：在成功初始连接时发送
- `DISCONNECTED`：在这种情况下，无法再检索数据，数据将向系统指示无法执行任何操作。可能的条件包括：
  - 指定的合约错误
  - 历史下载中断
  - 超过与 TWS 的重连次数
- `CONNBROKEN`：与 TWS 或数据农场的连接已丢失。数据馈送将尝试（通过存储）重新连接并在需要时回填，并恢复操作。
- `NOTSUBSCRIBED`：合约和连接正常，但由于缺乏权限无法检索数据。数据将向系统指示无法检索

数据。
- `DELAYED`：表示历史/回填操作正在进行中，策略处理的数据不是实时数据。
- `LIVE`：表示策略从这一点开始处理的数据是实时数据。

策略开发人员应考虑在发生断开连接或接收到延迟数据时应采取哪些措施。

### 数据时间框架和压缩

backtrader 生态系统中的数据馈送在创建期间支持 `timeframe` 和 `compression` 参数。这些参数也可以作为属性访问，使用 `data._timeframe` 和 `data._compression`。

时间框架/压缩组合的意义在于将数据传递给 cerebro 实例，通过 `resampledata` 或 `replaydata` 让内部重采样器/重放器对象了解目标是什么。 `_timeframe` 和 `_compression` 将在重采样/重放期间在数据中被覆盖。

但在实时数据馈送中，这些信息可能起重要作用。请看以下示例：

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

### 时间管理

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

### 实时数据馈送和重采样/重放

设计决策关于何时为实时数据馈送交付条：

尽可能实时地交付它们。这似乎是显而易见的，对于 Ticks 时间框架来说确实如此，但如果重采样/重放起作用，可能会有延迟。用例：

重采样配置为 Seconds/5：

```python
cerebro.resampledata(data, timeframe=bt.TimeFrame.Seconds, compression=5)
```

一个时间戳为 23:05:27.325000 的 tick 被传递。

市场交易缓慢，下一 tick 在 23:05:59.025000 传递。

这可能并不显而易见，但 backtrader 不知道交易非常慢，下一 tick 将在大约 32 秒后到来。如果没有任何规定，带时间戳 23:05:30.000000 的重采样条将延迟大约 29 秒。

这就是为什么实时数据馈送每 x 秒（浮动值）唤醒一次，通知重采样器/重放器没有新数据进来。这是通过在创建实时数据馈送时使用参数 `qcheck`（默认值：0.5 秒）来控制的。

这意味着重采样器每 `qcheck` 秒有一次机会交付条，如果本地时钟说重采样周期结束。这样，前述场景中的重采样条（23:05:30.000000）将最多在报告时间后的 `qcheck` 秒内交付。

由于默认值是 0.5 秒，最迟时间将是：23:05:30.500000。这比之前早了将近 29 秒。

缺点是：

一些 tick 可能太晚，无法用于已经交付的重采样/重放条。如果在交付之后，TWS 从服务器收到一个时间戳为 23:05:29.995000 的延迟消息，这对于已经报告给系统的时间 23:05.30.000000 来说太晚了。

这种情况主要发生在：

`timeoffset` 在 IBStore 中被禁用（设置为 False）且 IB 报告的时间和本地时钟之间的时间差显著时。

避免大多数这些延迟样本的最佳方法：

增加 `qcheck` 值，以允许考虑延迟消息：

```python
data = ibstore.getdata('TWTR', qcheck=2.0, ...)
```

这应该增加额外的空间，即使它会延迟重采样/重放条的交付。

**注意**：当然，对于 Seconds/5 重采样来说，2.0 秒的延迟与 Minutes/10 的重采样有不同的意义。

如果出于某种原因，最终用户希望禁用 `timeoffset` 并不通过 `qcheck` 管理，仍然可以获取延迟样本：

使用 `_latethrough` 设置为 True 作为 `getdata / IBData` 的参数：

```python
data = ibstore.getdata('TWTR', _latethrough=True, ...)
```

在重采样/重放时使用 `takelate` 设置为 True：

```python
cerebro.resampledata(data, takelate=True)
```

### IBBroker - 实时交易

**注意**：应要求，在 backtrader 中提供的模拟经纪商中实现了 `tradeid` 功能。这允许正确分配不同 `tradeid` 的佣金。

由于经纪商在报告佣金时，不支持此实时经纪商，因为佣金是在不可能分离不同 `tradeid` 值的时间点报告的。

尽管仍可以指定 `tradeid`，但它不再有意义。

#### 使用经纪商

要使用 IB 经纪商，必须替换 cerebro 创建的标准模拟经纪实例。

使用存储模型（推荐）：

```python
import backtrader as bt

cerebro = bt.Cerebro()
ibstore = bt.stores.IBStore(host='127.0.0.1', port=7496, clientId=35)
cerebro.broker = ibstore.getbroker()  # 或者 cerebro.setbroker(...)
```

使用直接

方法：

```python
import backtrader as bt

cerebro = bt.Cerebro()
cerebro.broker = bt.brokers.IBBroker(host='127.0.0.1', port=7496, clientId=35)
```

#### 经纪商参数

无论是直接还是通过 `getbroker`，IBBroker 都不支持任何参数。这是因为经纪商只是一个代理，真实的经纪商提供的内容不应被删除。

#### 一些限制

##### 现金和价值报告

内部 backtrader 模拟经纪商在调用策略 `next` 方法之前计算值（净清算价值）和现金，而实时经纪商不能保证这一点。

如果请求这些值，`next` 的执行可能会延迟到答案到达。

经纪商可能尚未计算这些值。

backtrader 告诉 TWS 提供更新后的值，但不知道消息何时会到达。

`IBBroker` 的 `getcash` 和 `getvalue` 方法报告的值始终是从 IB 接收到的最新值。

**注意**：进一步的限制是，这些值以账户的基本货币报告，即使可用更多货币的值。这是一个设计选择。

##### 头寸

backtrader 使用 TWS 报告的资产的头寸（价格和数量）。可以通过订单执行和订单状态消息进行内部计算，但如果丢失了这些消息中的一些，计算将不准确。

当然，如果在连接到 TWS 时，将进行交易的资产已经有一个未平仓头寸，策略计算的交易将无法像往常一样工作，因为有一个初始偏移。

#### 交易

在使用方面没有变化。只需使用策略中提供的方法（请参阅策略参考以获取完整说明）：

- `buy`
- `sell`
- `close`
- `cancel`

#### 返回的订单对象

与 backtrader 订单对象兼容（在相同层次结构中子类化）。

#### 订单执行类型

IB 支持众多执行类型，其中一些由 IB 模拟，一些由交易所支持。决定最初支持哪些订单执行类型的动机是：

与 backtrader 中提供的模拟经纪商兼容

理由是将进行生产的内容已经过回测。

因此，订单执行类型限于模拟经纪商中可用的类型：

- `Order.Market`
- `Order.Close`
- `Order.Limit`
- `Order.Stop`（触发止损后跟随一个市价单）
- `Order.StopLimit`（触发止损后跟随一个限价单）

**注意**：IB 根据不同策略触发止损。backtrader 不会修改默认设置，即 0：

0 - 默认值。对于 OTC 股票和美国期权，将使用“双重买/卖”方法。所有其他订单将使用“最后”方法。
如果用户希望修改此设置，可以按照 IB 文档提供额外的 **kwargs。 例如，在策略的 `next` 方法中：

```python
def next(self):
    # 一些逻辑
    self.buy(data, m_triggerMethod=2)
```

这将策略改为 2（“最后”方法，其中止损订单基于最后价格触发）。

请参考 IB API 文档以获取关于止损触发的进一步说明。

#### 订单有效期

backtrader 在回测期间可用的相同有效期概念（使用 `valid` 参数买入和卖出）在此也可用，并具有相同含义。因此，有效期参数对于 IB 订单转换如下：

- `None` -> `GTC`（Good Til Cancelled）

  因为未指定有效期，理解为订单必须有效直到取消。

- `datetime/date` -> `GTD`（Good Til Date）

  传递 `datetime.datetime` 或 `datetime.date` 实例表示订单必须有效直到某个时间点。

- `timedelta(x)` -> `GTD`（`timedelta(x)` 不等于 `timedelta()`）

  解释为订单有效从现在起加上 `timedelta(x)`。

- `float` -> `GTD`

  如果值取自 backtrader 使用的原始浮点日期时间存储，则订单必须有效直到由该浮点数指示的日期时间。

- `timedelta()` 或 0 -> `DAY`

  指定了一个值（而不是 `None`），但为空值，解释为当天（会话）有效的订单。

#### 通知

标准订单状态将通过策略的 `notify_order` 方法通知（如果覆盖）。

- `Submitted` - 订单已发送到 TWS
- `Accepted` - 订单已被放置
- `Rejected` - 订单放置失败或在其生命周期内被系统取消
- `Partial` - 部分执行已发生
- `Completed` - 订单已完全执行
- `Canceled`（或 `Cancelled`）

  在 IB 中有多重含义：

  - 用户手动取消
  - 服务器/交易所取消订单
  - 订单有效期已过

  将应用一个启发式方法，如果从 TWS 收到 `openOrder` 消息，订单状态指示 `PendingCancel` 或 `Canceled`，则订单将被标记为已过期。

- `Expired` - 请参见上文解释。

### 参考

#### IBStore

```python
class backtrader.stores.IBStore()
```

包装一个 ibpy `ibConnection` 实例的单例类。

这些参数也可以在使用该存储的类中指定，如 `IBData` 和 `IBBroker`。

参数：

- `host`（默认：`127.0.0.1`）：IB TWS 或 IB Gateway 实际运行的位置。虽然这通常是 localhost，但不一定是。
- `port`（默认：7496）：连接到的端口。演示系统使用 7497。
- `clientId`（默认：无）：用于连接 TWS 的 clientId。
  - 无：生成 1 到 65535 之间的随机 id
  - 整数：将传递为使用的值。
- `notifyall`（默认：False）
  - 如果为 False，则只会将错误消息发送到 Cerebro 和 Strategy 的 `notify_store` 方法。
  - 如果为 True，将通知收到的每一条来自 TWS 的消息。
- `_debug`（默认：False）
  - 将所有收到的 TWS 消息打印到标准输出。
- `reconnect`（默认：3）
  - 初始连接尝试失败后尝试重新连接的次数。
  - 设置为 -1 值以无限期重新连接。
- `timeout`（默认：3.0）
  - 重新连接尝试之间的时间（秒）。
- `timeoffset`（默认：True）
  - 如果为 True，将使用从 `reqCurrentTime`（IB 服务器时间）获得的时间计算与本地时间的偏移量，并将该偏移量用于价格通知（例如，CASH 市场的 `tickPrice` 事件）以修改本地计算的时间戳。
  - 时间偏移将传播到 backtrader 生态系统的其他部分，如重采样以使用计算的偏移量对齐重采样时间戳。
- `timerefresh`（默认：60.0）
  - 时间（秒）：刷新时间偏移的频率。
- `indcash`（默认：True）
  - 将 IND 代码管理为现金以获取价格。

#### IBBroker

```python
class backtrader.brokers.IBBroker(**kwargs)
```

互动经纪商的经纪商实现。

该类将互动经纪商的订单/头寸映射到 backtrader 的内部 API。

注意事项：
- `tradeid` 实际上不受支持，因为盈亏是直接从互动经纪商获取的。因为（如预期）以 FIFO 方式计算，因此 `tradeid` 的盈亏不准确。
- 头寸：如果在操作开始时有资产的未平仓头寸，或者通过其他方式给出的订单改变了头寸，Cerebro 中策略计算的交易将不反映现实。
  - 为避免这种情况，经纪商将不得不进行自己的头寸管理，这也将允许多个 id 的 `tradeid`（盈亏也将本地计算），但可能被认为违背了与实时经纪商合作的目的。

#### IBData

```python
class backtrader.feeds.IBData(**kwargs)
```

互动经纪商数据馈送。

支持以下参数 `dataname` 中的合约规格：

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
  - 如果在 `dataname` 规范中未提供，应用的默认值为证券类型。
- `exchange`（默认：SMART）
  - 如果在 `dataname` 规范中未提供，应用的默认值为交易所。
- `currency`（默认：''）
  - 如果在 `dataname` 规范中未提供，应用的默认值为货币。
- `historical`（默认：False）
  - 如果设置为 True，数据馈送将在首次下载数据后停止。
  - 将使用标准数据馈送参数 `fromdate` 和 `todate` 作为参考。
  - 如果请求的持续时间大于 IB 在给定时间框架/压缩组合下允许的持续时间，数据馈送将进行多次请求。
- `what`（默认：None）
  - 如果为 None，则历史数据请求的默认值将根据资产类型的不同而不同：
    - CASH 资产为 'BID'
    - 其他为 'TRADES'
  - 如果希望其他值，请检查 IB API 文档。
- `rtbar`（默认：False）
  - 如果为 True，将使用互动经纪商提供的 5 秒实时条作为最小刻度。根据文档，这些条形图对应于实时值（经过 IB 整理和处理后）。
  - 如果为 False，则将使用 RTVolume 价格，这些价格基于接收的 tick。在 CASH 资产（如 EUR.JPY）的情况下，始终将使用 RTVolume，并从中提取买入价格（根据互联网上的文献，这已成为与 IB 的行业默认标准）。
  - 即使设置为 True，如果数据重采样/保留到低于 Seconds/5 的时间框架/压缩，也不会使用实时条，因为 IB 不会在该级别以下提供它们。
- `qcheck`（默认：0.5）
  - 在没有数据接收时唤醒的时间（秒），以便适当地重采样/重放数据包并将通知上传到链中。
- `backfill_start`（默认：True）
  - 在启动时执行回填。将通过单个请求获取最大可能的历史数据。
- `backfill`（默认：True）
  - 在断开/重新连接周期后执行回填。将使用间隙持续时间下载最小可能的数据量。
- `backfill_from`（默认：None）
  - 可以传递另一个数据源以进行初始回填。一旦数据源耗尽，如果需要，将从 IB 回填。理想情况下，这意味着从已存储的来源（如磁盘上的文件）进行回填，但不限于此。
- `latethrough`（默认：False）
  - 如果数据源被重采样/重放，一些 tick 可能太晚，无法用于已经交付的重采样/重放条。如果为 True，将允许这些 tick 通过。
  - 请检查重采样器文档以了解如何考虑这些 tick。
  - 特别是当 `timeoffset` 在 IBStore 实例中设置为 False 且 TWS 服务器时间与本地计算机不同步时，可能会发生这种情况。
- `tradename`（默认：None）
  - 对于某些特定情况（如 CFD），有时价格由一个资产提供，交易在另一个资产中进行。

  ```python
  SPY-STK-SMART-USD -> SP500 ETF（将指定为 `dataname`）
  SPY-CFD-SMART-USD -> 对应的 CFD，不提供价格跟踪，但在这种情况下将是交易资产（指定为 `tradename`）。
  ```

参数中的默认值允许像 `TICKER` 这样的情况，参数 `sectype`（默认：STK）和 `exchange`（默认：SMART）适用。

一些资产（如 AAPL）需要包括货币在内的完整规格（默认：''），而其他资产（如 TWTR）可以直接传递。

AAPL-STK-SMART-USD 将是 `dataname` 的完整规格。

或者：IBData 作为 `IBData(dataname='AAPL', currency='USD')` 使用默认值（STK 和 SMART）并覆盖货币为 USD。
