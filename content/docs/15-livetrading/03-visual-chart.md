---
title: "Visual Chart"
weight: 3
---

# Visual Chart

Visual Chart 的集成支持以下功能：

- 实时数据馈送
- 实时交易

Visual Chart 是一个完整的交易解决方案：

- 集成图表、数据馈送和经纪功能于单一平台

有关更多信息，请访问：[www.visualchart.com](http://www.visualchart.com)

## 要求

- **VisualChart 6**（运行在 Windows 上）
- **comtypes fork**： [https://github.com/mementum/comtypes](https://github.com/mementum/comtypes)

可以通过以下命令安装：

```bash
pip install https://github.com/mementum/comtypes/archive/master.zip
```

Visual Chart 的 API 基于 COM。目前 comtypes 主分支不支持解包 VT_ARRAYS of VT_RECORD。这是 Visual Chart 所使用的。Pull Request #104 已提交，但尚未集成。一旦集成，可以使用主分支。

- **pytz**（可选但强烈推荐）：确保每个数据都在市场时间返回。这对于大多数市场来说都是正确的，但有些市场确实是例外（全球指数就是一个很好的例子）。

## 示例代码

源代码中包含完整示例：`samples/vctest/vctest.py`。

## VCStore - 存储

存储是实时数据馈送/交易支持的核心，提供了 COM API 和数据馈送及经纪代理之间的适配层。

可以通过以下方法获取经纪商实例：
```python
VCStore.getbroker(*args, **kwargs)
```

可以通过以下方法获取数据馈送实例：
```python
VCStore.getdata(*args, **kwargs)
```
在这种情况下，许多 **kwargs 是数据馈送的常见参数，如 dataname、fromdate、todate、sessionstart、sessionend、timeframe、compression。

VCStore 将尝试：
- 使用 Windows 注册表自动定位 VisualChart 在系统中的位置。
- 如果找到，将扫描安装目录中的 COM DLLs 以创建 COM typelibs 并实例化适当的对象。
- 如果未找到，将尝试使用已知和硬编码的 CLSIDs 进行相同操作。

注意：即使通过扫描文件系统找到 DLLs，Visual Chart 本身也必须在运行。backtrader 不会启动 Visual Chart。

## VCData feeds

### 一般
Visual Chart 提供的数据馈送具有一些有趣的属性：
- 重新采样由平台完成
- 并非所有情况下：秒不支持，仍需由 backtrader 完成

例如：

```python
vcstore = bt.stores.VCStore()
vcstore.getdata(dataname='015ES', timeframe=bt.TimeFrame.Ticks)
cerebro.resampledata(data, timeframe=bt.TimeFrame.Seconds, compression=5)
```

大多数情况下，只需以下操作：

```python
vcstore = bt.stores.VCStore()
data = vcstore.getdata(dataname='015ES', timeframe=bt.TimeFrame.Minutes, compression=2)
cerebro.adddata(data)
```

数据将通过比较内部设备时钟和平台提供的 tick 来计算内部时间偏移，以便在没有新 tick 进入时尽早传递自动重新采样的 bar。

### 实例化数据

按 VisualChart 左上角显示的符号（无空格）传递。例如：

ES-Mini 显示为 001 ES。实例化为：
```python
data = vcstore.getdata(dataname='001ES', ...)
```

EuroStoxx 50 显示为 015 ES。实例化为：
```python
data = vcstore.getdata(dataname='015ES', ...)
```

注意：backtrader 会努力清除第四位置的空格（如果名称直接从 Visual Chart 粘贴）。

## 时间管理

时间管理遵循 backtrader 的一般规则。为了确保代码不依赖于 DST 转换，请使用市场时间。

## 数据通知

数据馈送将通过以下一种或多种方式报告当前状态（检查 Cerebro 和策略参考）：
- `Cerebro.notify_data`（如果覆盖）
- `Cerebro.adddatacb` 添加的回调
- `Strategy.notify_data`（如果覆盖）

策略中的示例：
```python
class VCStrategy(bt.Strategy):
    def notify_data(self, data, status, *args, **kwargs):
        if status == data.LIVE:  # 数据已切换到实时数据
            # 做某些事
            pass
```

以下通知将根据系统中的更改发送：
- `CONNECTED`：成功初始连接时发送
- `DISCONNECTED`：无法检索数据时发送
- `CONNBROKEN`：连接丢失时发送
- `NOTSUBSCRIBED`：无权限检索数据时发送
- `DELAYED`：历史/回填操作正在进行时发送
- `LIVE`：数据已切换到实时数据时发送

## VCBroker - 实时交易

### 使用经纪商
要使用 VCBroker，需要替换 cerebro 创建的标准经纪商模拟实例。

使用存储模型（推荐）：
```python
import backtrader as bt

cerebro = bt.Cerebro()
vcstore = bt.stores.VCStore()
cerebro.broker = vcstore.getbroker()  # 或 cerebro.setbroker(...)
```

### 经纪商参数
VCBroker 不支持任何参数，因为经纪商只是实际经纪商的代理。实际经纪商提供的功能不会被去除。

## 限制

### 头寸
Visual Chart 报告未平仓头寸，但缺少头寸已关闭的最终事件。因此，backtrader 需要完全记录头寸，并与账户中的任何先前头寸分开。

### 佣金
COM 交易接口不报告佣金。除非在实例化经纪商时提供了表示实际佣金的 `Commission` 实例，否则 backtrader 无法准确估算佣金。

## 交易操作

使用方面没有变化。只需使用策略中提供的方法（请参阅策略参考以获取完整说明）：
- `buy`
- `sell`
- `close`
- `cancel`

## 订单执行类型
Visual Chart 支持 backtrader 需要的最小订单执行类型，因此任何经过回测的内容都可以上线。限于：
- `Order.Market`
- `Order.Close`
- `Order.Limit`
- `Order.Stop`（触发 Stop 时跟随市价单）
- `Order.StopLimit`（触发 Stop 时跟随限价单）

## 订单有效期

backtrader 在回测期间可用的相同有效期概念（使用 `valid` 参数买入和卖出）在此也可用，并具有相同含义。

## 通知
标准订单状态将通过策略的 `notify_order` 方法通知（如果覆盖）：
- `Submitted` - 订单已发送到 TWS
- `Accepted` - 订单已被放置
- `Rejected` - 订单放置失败或在其生命周期内被系统取消
- `Partial` - 已部分执行
- `Completed` - 订单已完全执行
- `Canceled`（或 `Cancelled`）

## 参考

### VCStore

```python
class backtrader.stores.VCStore()
```
封装 ibpy ibConnection 实例的单例类。

### VCBroker

```python
class backtrader.brokers.VCBroker(**kwargs)
```
VisualChart 的经纪商实现。

### VCData

```python
class backtrader.feeds.VCData(**kwargs)
```
VisualChart 数据馈送。

参数：
- `qcheck`（默认：0.5）：默认唤醒超时以让重采样/重放器检查当前 bar 是否可以交付。
- `historical`（默认：False）：如果未提供 `todate` 参数（在基类中定义），则如果设置为 True，将强制仅进行历史下载。
- `milliseconds`（默认：True）：Visual Chart 构建的 bar 具有以下格式：HH:MM:59.999000。如果该参数为 True，将添加一毫秒，使其显示为：HH:MM:59.999000。
- `tradename`（默认：无）：无法交易连续期货，但它们非常适合数据跟踪。如果提供该参数，它将是当前期货的名称，该期货将是交易资产。
- `usetimezones`（默认：True）：对于大多数市场，Visual Chart 提供的时间偏移信息允许将日期时间转换为市场时间。

