---
title: "跨平台回测的陷进"
weight: 2
---

### 跨平台回测的陷阱

在 Backtrader 社区中，经常有用户希望将 TradingView 等流行的回测平台上的回测结果进行复制。TradingView 使用的脚本语言是 Pinescript，而用户往往并不了解该语言的具体实现，也未接触过回测引擎的内部机制。因此，即使用户有意复制回测结果，也必须明白跨平台编程有其局限性。

#### 指标：并不总是忠实于原始定义
当在 Backtrader 中实现新指标时，开发者会特别强调尊重指标的原始定义。例如，RSI 指标就是一个典型的例子。

Welles Wilder 设计 RSI 时使用了修改过的移动平均（即平滑移动平均，参见 Wikipedia - Modified Moving Average）。然而，许多平台提供的 RSI 指标，实际上使用的是经典的指数移动平均（EMA），而非书中的定义。

尽管两者的差别并不算巨大，但这并不是 Wilder 原始定义的 RSI。它可能仍然有用，甚至可能更好，但它并不等同于 Wilder 所定义的 RSI。而且，大多数文档（如果有的话）并未提到这一点。

在 Backtrader 中，RSI 的默认配置使用 MMA，以保持忠实于原始定义。不过，开发者可以通过子类化或者在运行时实例化时，选择使用 EMA 或者简单移动平均（SMA）来替代。

#### 例子：唐奇安通道
Wikipedia 中的定义是这样的：**Wikipedia - Donchian Channel**。但是，它只是一些文字，未提到如何使用通道突破作为交易信号。

另外，以下两个定义明确说明，计算通道时数据不包括当前的柱线，因为如果包括了，突破就无法被反映出来：
- **StockCharts - School - Price Channels**
- **IncredibleCharts - Donchian Channels**

这些来源明确指出，计算通道时不包含当前的价格柱线，这样突破才会被正确显示。以下是来自 StockCharts 的示例图表：

**StockCharts - Donchian Channels - Breakouts**

然后，我们看看 TradingView。首先是链接：
**TradingView - Wiki - Donchian Channels**

以及该页面的图表：

**TradingView - Donchian Channels - No Breakouts**

甚至 Investopedia 也使用了来自 TradingView 的图表，显示没有突破：

**Investopedia - Donchian Channels**

许多人可能会惊讶，因为 TradingView 中没有显示突破。这意味着 TradingView 的 Donchian 通道实现方式将当前的价格柱线也考虑进了计算。

#### Backtrader 中的唐奇安通道
Backtrader 中没有内置的 DonchianChannels 实现，但我们可以轻松地创建一个。是否将当前柱线用于通道计算，是一个可以调节的参数。

以下是代码示例：

```python
class DonchianChannels(bt.Indicator):
    '''
    参数说明：
      - ``lookback``（默认：-1）
      
        如果是 `-1`，则考虑从过去一根柱线开始计算，当前的高/低价可能突破通道。
        
        如果是 `0`，则当前的价格将被用于计算 Donchian 通道。这意味着价格**永远**不会突破上下通道带。
    '''

    alias = ('DCH', 'DonchianChannel',)

    lines = ('dcm', 'dch', 'dcl',)  # dc 中线，dc 高线，dc 低线
    params = dict(
        period=20,
        lookback=-1,  # 是否考虑当前柱线
    )

    plotinfo = dict(subplot=False)  # 与数据一起绘制
    plotlines = dict(
        dcm=dict(ls='--'),  # 虚线
        dch=dict(_samecolor=True),  # 使用与前一个线条相同的颜色（dcm）
        dcl=dict(_samecolor=True),  # 使用与前一个线条相同的颜色（dch）
    )

    def __init__(self):
        hi, lo = self.data.high, self.data.low
        if self.p.lookback:  # 根据需要向后移动
            hi, lo = hi(self.p.lookback), lo(self.p.lookback)

        self.l.dch = bt.ind.Highest(hi, period=self.p.period)
        self.l.dcl = bt.ind.Lowest(lo, period=self.p.period)
        self.l.dcm = (self.l.dch + self.l.dcl) / 2.0  # 上下通道的平均值
```

使用 `lookback=-1` 的配置，生成的图表如下（放大查看）：

**Backtrader - Donchian Channels - Breakouts**

可以清晰地看到突破，而使用 `lookback=0` 时则看不到突破。

**Backtrader - Donchian Channels - No Breakouts**

#### 编码中的隐含问题
如果程序员首先在商业平台上实现策略，并且由于图表未显示突破，他们就可能会进行如下的代码编写：

```python
if price0 > channel_high_1:
    sell()
elif price0 < channel_low_1:
    buy()
```

其中，`price0` 会与前一周期的通道高/低进行比较（因此 `_1` 后缀表示前一周期）。

然而，在不了解 Backtrader 中 Donchian 通道默认包含突破的情况下，程序员会编写如下代码：

```python
def __init__(self):
    self.donchian = DonchianChannels()

def next(self):
    if self.data[0] > self.donchian.dch[-1]:
        self.sell()
    elif self.data[0] < self.donchian.dcl[-1]:
        self.buy()
```

这是错误的！因为突破是在进行比较的同时就已经发生了。正确的代码应该是：

```python
def __init__(self):
    self.donchian = DonchianChannels()

def next(self):
    if self.data[0] > self.donchian.dch[0]:
        self.sell()
    elif self.data[0] < self.donchian.dcl[0]:
        self.buy()
```

虽然这只是一个小例子，但它揭示了回测结果如何因指标实现的差异而产生不同的情况。虽然差别看起来不大，但错误的交易判断可能会带来巨大的影响。
