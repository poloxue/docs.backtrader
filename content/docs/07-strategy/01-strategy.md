---
title: "策略类 Strategy"
weight: 1
---

### 策略

在 backtrader 中，Cerebro 实例是整个系统的核心，而策略是平台用户的核心。

#### 策略的生命周期方法

**注意**  
策略可以在创建时通过抛出 `StrategySkipError` 异常来中断，该异常来自 `backtrader.errors` 模块。这将避免在回测期间处理该策略。请参阅“异常”部分。

**构建：`__init__`**

这是在实例化期间调用的：指标将在此处创建以及其他需要的属性。例如：

```python
def __init__(self):
    self.sma = btind.SimpleMovingAverage(period=15)
```

**启动：`start`**

Cerebro 实例通知策略是时候开始运行了。存在一个默认的空方法。

**初期：`prenext`**

在创建期间声明的指标将对策略的成熟期施加限制：这称为最小周期。上面的 `__init__` 创建了一个周期为 15 的简单移动平均线 (SMA)。

只要系统看到的 bar 少于 15 个，就会调用 `prenext`（默认实现为空操作）。

**成熟：`next`**

一旦系统看到 15 个 bar 并且 SMA 有足够的缓冲区开始生成值，策略就足够成熟可以真正执行。

存在一个 `nextstart` 方法，会在从 `prenext` 切换到 `next` 时调用一次。`nextstart` 的默认实现是简单地调用 `next`。

**繁衍：无**

策略实际上不会繁衍，但从某种意义上来说，它们会，因为系统会在优化时实例化它们多次（使用不同的参数）。

**结束：`stop`**

系统通知策略是时候重置并整理一切了。存在一个默认的空方法。

通常情况下和常规使用模式下，这看起来像这样：

```python
class MyStrategy(bt.Strategy):

    def __init__(self):
        self.sma = btind.SimpleMovingAverage(period=15)

    def next(self):
        if self.sma > self.data.close:
            # 执行某些操作
            pass
        elif self.sma < self.data.close:
            # 执行其他操作
            pass
```

在这个代码片段中：

- 在 `__init__` 中分配一个指标给属性
- 默认的空 `start` 方法未被重写
- `prenext` 和 `nextstart` 未被重写
- 在 `next` 中，指标的值与收盘价进行比较，以执行某些操作
- 默认的空 `stop` 方法未被重写

#### 策略事件通知

策略将每个 `next` 周期收到通知：

- 通过 `notify_order(order)` 接收任何订单状态的变化通知
- 通过 `notify_trade(trade)` 接收任何交易的开启/更新/关闭通知
- 通过 `notify_cashvalue(cash, value)` 接收经纪账户中的当前现金和投资组合
- 通过 `notify_fund(cash, value, fundvalue, shares)` 接收经纪账户中的当前现金、投资组合、基金价值和份额
- 通过 `notify_store(msg, *args, **kwargs)` 接收来自存储提供者的通知

#### 如何买入/卖出/平仓

`buy` 和 `sell` 方法生成订单。调用这些方法时，它们返回一个 Order（或其子类）实例，该实例可用作引用。此订单具有唯一的 `ref` 标识符，可用于比较。

**注意**  
特定经纪实现的 Order 子类可能携带经纪提供的其他唯一标识符。

要创建订单，请使用以下参数：

- `data`（默认：None）：创建订单的数据。如果为 None，则使用系统中的第一个数据 `self.datas[0]` 或 `self.data0`（即 `self.data`）。
- `size`（默认：None）：要使用的订单数量（正值）。如果为 None，则使用通过 `getsizer` 检索到的 `sizer` 实例来确定大小。
- `price`（默认：None）：使用的价格（实时经纪可能对格式有最低价格步长要求）。对于市场和收盘订单（Market 和 Close orders）为 None（市场决定价格）。对于限价、止损和止损限价订单，这个值决定触发点。
- `plimit`（默认：None）：仅适用于止损限价订单。这是在触发止损后设置隐含限价订单的价格（使用 `price`）。
- `exectype`（默认：None）：可能的值：
  - `Order.Market` 或 None：市场订单将以下一个可用价格执行。在回测中，将是下一个 bar 的开盘价。
  - `Order.Limit`：订单只能以给定价格或更好价格执行。
  - `Order.Stop`：订单在 `price` 触发时被触发，并像 `Order.Market` 订单一样执行。
  - `Order.StopLimit`：订单在 `price` 触发时被触发，并作为隐含的限价订单执行，限价由 `pricelimit` 给出。
- `valid`（默认：None）：可能的值：
  - None：生成一个不会过期的订单（即 Good til cancel），并保留在市场上直到匹配或取消。实际上，经纪通常会施加一个时间限制，但通常远远在未来，可以认为它不会过期。
  - `datetime.datetime` 或 `datetime.date` 实例：日期将用于生成有效期至给定日期的订单（即 good til date）。
  - `Order.DAY` 或 0 或 `timedelta()`：生成一个有效期至交易日结束的订单（即日订单）。
  - 数值：假定为 `matplotlib` 编码的 `datetime` 值（`backtrader` 使用的），将用于生成有效期至该时间的订单（即 good til date）。
- `tradeid`（默认：0）：`backtrader` 用于跟踪同一资产上重叠交易的内部值。当通知订单状态变化时，此 `tradeid` 会发送回策略。

**示例**：如果 `backtrader` 直接支持的 4 种订单执行类型不够，可以为 Interactive Brokers 传递以下参数：

```python
orderType='LIT', lmtPrice=10.0, auxPrice=9.8
```

这将覆盖 `backtrader` 创建的设置，并生成一个 `LIMIT IF TOUCHED` 订单，触及价格为 9.8，限价为 10.0。

### 策略信息：

- `env`：策略所属的 cerebro 实体
- `datas`：传递给 cerebro 的数据源数组
- `data/data0`：`datas[0]` 的别名
- `dataX`：`datas[X]` 的别名
- `dnames`：按名称访问数据源的替代方法（通过 [name] 或 .name 语法）
- `broker`：与此策略关联的经纪（从 cerebro 接收）
- `stats`：包含 cerebro 为此策略创建的观察者的列表/命名元组序列
- `analyzers`：包含 cerebro 为此策略创建的分析器的列表/命名元组序列
- `position`：获取 `data0` 的当前仓位的属性

### 成员属性（用于统计/观察者/分析器）：

- `_orderspending`：将在调用 `next` 之前通知策略的订单列表
- `_tradespending`：将在调用 `next` 之前通知策略的交易列表
- `_orders`：已经通知的订单列表。订单可以在列表中多次出现，具有不同的状态和执行部分。列表旨在保留历史记录。
- `_trades`：已经通知的交易列表。交易可以像订单一样多次出现在列表中。

**注意**  
`prenext`、`nextstart` 和 `next` 可以针对同一时间点多次调用（例如，当使用每日时间框架时，价格更新每日 bar 的 ticks）。

### 策略参考
`class backtrader.Strategy(*args, **kwargs)`

基类，用于子类化用户定义的策略。

#### 方法

- `next()`：当所有数据/指标的最小周期满足时，将为所有剩余数据点调用此方法。
- `nextstart()`：当所有数据/指标的最小周期满足时，将调用一次此方法。默认行为是调用 `next`。
- `prenext()`：在满足所有数据/指标的最小周期之前调用此方法。
- `start()`：在回测即将开始之前调用。
- `stop()`：在回测即将结束之前调用。
- `notify_order(order)`：当订单状态变化时接收通知。
- `notify_trade(trade)`：当交易状态变化时接收通知。
- `notify_cashvalue(cash, value)`：接收策略经纪账户的当前资金和投资组合状态。
- `notify_fund(cash, value, fundvalue, shares)`：接收当前现金、投资组合、基金价值和份额。
- `notify_store(msg, *args, **kwargs)`：接收存储提供者的

通知。

#### 订单方法

- `buy(...)`：创建买入订单并发送给经纪。
- `sell(...)`：创建卖出（空头）订单并发送给经纪。
- `close(...)`：关闭现有仓位。
- `cancel(order)`：取消经纪中的订单。

#### 其他方法

- `buy_bracket(...)`：创建括号订单组（低侧 - 买入订单 - 高侧）。
- `sell_bracket(...)`：创建括号订单组（低侧 - 卖出订单 - 高侧）。
- `order_target_size(...)`：下订单将仓位重新平衡为目标大小。
- `order_target_value(...)`：下订单将仓位重新平衡为目标价值。
- `order_target_percent(...)`：下订单将仓位重新平衡为当前投资组合价值的目标百分比。
- `getsizer()`：返回用于自动计算头寸的 `sizer`。
- `setsizer(sizer)`：替换默认的 `sizer`。
- `getsizing(data=None, isbuy=True)`：返回 `sizer` 实例为当前情况计算的头寸。
- `getposition(data=None, broker=None)`：返回给定数据和经纪的当前仓位。
- `getpositionbyname(name=None, broker=None)`：按名称返回给定经纪的当前仓位。
- `getpositionsbyname(broker=None)`：直接从经纪获取按名称的所有仓位。
- `getdatanames()`：返回现有数据名称的列表。
- `getdatabyname(name)`：使用环境（cerebro）按名称返回给定数据。
- `add_timer(...)`：计划一个计时器以调用指定的回调或策略的 `notify_timer`。
- `notify_timer(timer, when, *args, **kwargs)`：接收计时器通知，其中 `timer` 是由 `add_timer` 返回的计时器，`when` 是调用时间。`args` 和 `kwargs` 是传递给 `add_timer` 的额外参数。
