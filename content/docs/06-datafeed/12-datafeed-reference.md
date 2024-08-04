---
title: "数据源参考"
weight: 12
---

### 数据源参考

#### AbstractDataBase

**数据行（Lines）:**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数（Params）:**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)

---

#### BacktraderCSVData

解析用于测试的自定义 CSV 数据。

**特定参数：**
- dataname: 要解析的文件名或类文件对象

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)

---

#### CSVDataBase

用于实现 CSV 数据源的基类。

该类负责打开文件、读取行并将其标记化。子类只需重写 `_loadline(tokens)` 方法。

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)

---

#### Chainer

用于链式连接数据的类。

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)

---

#### DataClone

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)

---

#### DataFiller

该类将使用基础数据源的信息填充数据中的空隙。

**参数：**
- `fill_price` (def: None): 如果为 None，将使用上一条数据的收盘价；否则使用传递的值（例如 ‘NaN’）
- `fill_vol` (def: NaN): 用于填充缺失数据的交易量
- `fill_oi` (def: NaN): 用于填充缺失数据的未平仓合约量

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- fill_price (None)
- fill_vol (nan)
- fill_oi (nan)

---

#### DataFilter

此类过滤给定数据源中的数据行。除了 DataBase 的标准参数外，它还接受 `funcfilter` 参数，该参数可以是任何可调用对象。

**逻辑：**
- `funcfilter` 将与基础数据源一起调用
- 它可以是任何可调用对象
- 返回值 True：当前数据源的数据行值将被使用
- 返回值 False：当前数据源的数据行值将被丢弃

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- funcfilter (None)

---

#### GenericCSVData

根据定义的参数解析 CSV 文件。

**特定参数（或特定含义）：**
- dataname: 要解析的文件名或类文件对象
- `lines` 参数（datetime, open, high …）取数值
- 值为 -1 表示 CSV 源中不存在该字段
- 如果 time 存在（参数 time >=0），源包含分开的日期和时间字段，将合并

**参数：**
- nullvalue: 如果缺少值（CSV 字段为空），将使用的值
- dtformat: 用于解析 datetime CSV 字段的格式。请参阅 python strptime/strftime 文档以了解格式。
- 如果指定了数值，它将按以下方式解释：
  - 1: 值为代表自 1970 年 1 月 1 日以来的秒数的 Unix 时间戳（int 型）
  - 2: 值为代表自 1970 年 1 月 1 日以来的秒数的 Unix 时间戳（float 型）
- 如果传递了一个可调用对象，它将接受一个字符串并返回一个 datetime.datetime 实例
- tmformat: 用于解析 time CSV 字段的格式（如果存在）（time 字段默认不存在）

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)
- nullvalue (nan)
- dtformat (%Y-%m-%d %H:%M:%S)
- tmformat (%H:%M:%S)
- datetime (0)
- time (-1)
- open (1)
- high (2)
- low (3)
- close (4)
- volume (5)
- openinterest (6)

---

#### IBData

**交互式经纪商数据源（Interactive Brokers Data Feed）**

支持参数 `dataname` 中的合约规格：

```txt
TICKER # 股票类型和 SMART 交易所
TICKER-STK # 股票和 SMART 交易所
TICKER-STK-EXCHANGE # 股票
TICKER-STK-EXCHANGE-CURRENCY # 股票
TICKER-CFD # CFD 和 SMART 交易所
TICKER-CFD-EXCHANGE # CFD
TICKER-CDF-EXCHANGE-CURRENCY # 股票
TICKER-IND-EXCHANGE # 指数
TICKER-IND-EXCHANGE-CURRENCY # 指数
TICKER-YYYYMM-EXCHANGE # 期货
TICKER-YYYYMM-EXCHANGE-CURRENCY # 期货
TICKER-YYYYMM-EXCHANGE-CURRENCY-MULT # 期货
TICKER-FUT-EXCHANGE-CURRENCY-YYYYMM-MULT # 期货
TICKER-YYYYMM-EXCHANGE-CURRENCY-STRIKE-RIGHT # 期权
TICKER-YYYYMM-EXCHANGE-CURRENCY-STRIKE-RIGHT-MULT # 期权
TICKER-FOP-EXCHANGE-CURRENCY-YYYYMM-STRIKE-RIGHT # 期权
TICKER-FOP-EXCHANGE-CURRENCY-YYYYMM-STRIKE-RIGHT-MULT # 期权
CUR1.CUR2-CASH-IDEALPRO # 外汇
TICKER-YYYYMMDD-EXCHANGE-CURRENCY-STRIKE-RIGHT # 期权
TICKER-YYYYMMDD-EXCHANGE-CURRENCY-STRIKE-RIGHT-MULT # 期权
TICKER-OPT-EXCHANGE-CURRENCY-YYYYMMDD-STRIKE-RIGHT # 期权
TICKER-OPT-EXCHANGE-CURRENCY-YYYYMMDD-STRIKE-RIGHT-MULT # 期权
```

**参数：**
- sectype (默认: STK)
- exchange (默认: SMART)
- currency (默认: '')
- historical (默认: False)
- what (默认: None)
- rtbar (默认: False)
- qcheck (默认: 0.5)
- backfill_start (默认: True)
- backfill (默认: True)
-

 backfill_from (默认: None)
- latethrough (默认: False)
- tradename (默认: None)

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.5)
- calendar (None)
- sectype (STK)
- exchange (SMART)
- currency ()
- rtbar (False)
- historical (False)
- what (None)
- useRTH (False)
- backfill_start (True)
- backfill (True)
- backfill_from (None)
- latethrough (False)
- tradename (None)

---

#### InfluxDB

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- host (127.0.0.1)
- port (8086)
- username (None)
- password (None)
- database (None)
- startdate (None)
- high (high_p)
- low (low_p)
- open (open_p)
- close (close_p)
- volume (volume)
- ointerest (oi)

---

#### MT4CSVData

解析 Metatrader4 历史中心导出的 CSV 文件。

**特定参数（或特定含义）：**
- dataname: 要解析的文件名或类文件对象
- 使用 GenericCSVData 并简单修改参数

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)
- nullvalue (nan)
- dtformat (%Y.%m.%d)
- tmformat (%H:%M)
- datetime (0)
- time (1)
- open (2)
- high (3)
- low (4)
- close (5)
- volume (6)
- openinterest (-1)

---

#### OandaData

**参数：**
- qcheck (默认: 0.5)
- historical (默认: False)
- backfill_start (默认: True)
- backfill (默认: True)
- backfill_from (默认: None)
- bidask (默认: True)
- useask (默认: False)
- includeFirst (默认: True)
- reconnect (默认: True)
- reconnections (默认: -1)
- reconntimeout (默认: 5.0)

支持的时间框架和压缩映射符合 OANDA API 开发者指南中的定义：

```python
(TimeFrame.Seconds, 5): 'S5',
(TimeFrame.Seconds, 10): 'S10',
(TimeFrame.Seconds, 15): 'S15',
(TimeFrame.Seconds, 30): 'S30',
(TimeFrame.Minutes, 1): 'M1',
(TimeFrame.Minutes, 2): 'M3',
(TimeFrame.Minutes, 3): 'M3',
(TimeFrame.Minutes, 4): 'M4',
(TimeFrame.Minutes, 5): 'M5',
(TimeFrame.Minutes, 10): 'M10',
(TimeFrame.Minutes, 15): 'M15',
(TimeFrame.Minutes, 30): 'M30',
(TimeFrame.Minutes, 60): 'H1',
(TimeFrame.Minutes, 120): 'H2',
(TimeFrame.Minutes, 180): 'H3',
(TimeFrame.Minutes, 240): 'H4',
(TimeFrame.Minutes, 360): 'H6',
(TimeFrame.Minutes, 480): 'H8',
(TimeFrame.Days, 1): 'D',
(TimeFrame.Weeks, 1): 'W',
(TimeFrame.Months, 1): 'M',
```

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.5)
- calendar (None)
- historical (False)
- backfill_start (True)
- backfill (True)
- backfill_from (None)
- bidask (True)
- useask (False)
- includeFirst (True)
- reconnect (True)
- reconnections (-1)
- reconntimeout (5.0)

---

#### PandasData

使用 Pandas DataFrame 作为数据源。

**参数：**
- nocase (默认: True) 列名匹配不区分大小写

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- nocase (True)
- datetime (None)
- open (-1)
- high (-1)
- low (-1)
- close (-1)
- volume (-1)
- openinterest (-1)

---

#### PandasDirectData

使用 Pandas DataFrame 作为数据源，直接迭代由 `itertuples` 返回的元组。

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- datetime (0)
- open (1)
- high (2)
- low (3)
- close (4)
- volume (5)
- openinterest (6)

---

#### Quandl

直接从 Quandl 服务器下载数据。

**特定参数（或特定含义）：**
- dataname: 要下载的代码（例如 'YHOO'）
- baseurl: 服务器 URL
- proxies: 指示下载时使用的代理的字典，例如 {‘http’: ‘http://myproxy.com’}
- buffered: 如果为 True，整个 socket 连接将在解析前缓存在本地
- reverse: Quandl 返回的值按降序排列（最新的在前）。如果为 True，请求将告诉 Quandl 以升序（最旧的在前）格式返回
- adjclose: 是否使用股息/拆股调整后的收盘价，并根据它调整所有值
- apikey: 如果需要，使用的 API 密钥
- dataset: 标识要查询的数据集的字符串。默认为 WIKI

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)
- reverse (True)
- adjclose (True)
- round (False)
- decimals (2)
- baseurl ([https://www.quandl.com/api/v3/datasets](https://www.quandl.com/api/v3/datasets))
- proxies ({})
- buffered (True)
- apikey (None)
- dataset (WIKI)

---

#### QuandlCSV

解析预先下载的 Quandl CSV 数据源（或如果符合 Quandl 格式，本地生成的 CSV 文件）。

**特定参数：**
- dataname: 要解析的文件名或类文件对象
- reverse (默认: False): 假设本地存储的文件在下载过程中已被反向
- adjclose (默认: True): 是否使用股息/拆股调整后的收盘价，并根据它调整所有值
- round (默认: False): 是否在调整收盘价后将值四舍五入到特定的小数位数
- decimals (默认: 2): 要四舍五入的小数位数

**数据行：**
-

 close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)
- reverse (False)
- adjclose (True)
- round (False)
- decimals (2)

---

#### RollOver

在满足条件时切换到下一个期货合约。

**参数：**
- checkdate (默认: None): 必须是具有以下签名的可调用对象：

```python
checkdate(dt, d):
```
- dt: datetime.datetime 对象
- d: 当前活动期货的数据源

**返回值：**
- True: 只要返回 True，就可以切换到下一个期货
- False: 不能进行切换

- checkcondition (默认: None): 注意：只有当 checkdate 返回 True 时才会调用此方法。如果为 None，将在内部评估为 True（执行切换）。否则，必须是具有以下签名的可调用对象：

```python
checkcondition(d0, d1):
```
- d0: 当前活动期货的数据源
- d1: 下一个到期的数据源

**返回值：**
- True: 切换到下一个期货
- False: 不能进行切换

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- checkdate (None)
- checkcondition (None)

---

#### SierraChartCSVData

解析 SierraChart 导出的 CSV 文件。

**特定参数（或特定含义）：**
- dataname: 要解析的文件名或类文件对象

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)
- nullvalue (nan)
- dtformat (%Y/%m/%d)
- tmformat (%H:%M:%S)
- datetime (0)
- time (-1)
- open (1)
- high (2)
- low (3)
- close (4)
- volume (5)
- openinterest (6)

---

#### VCData

VisualChart 数据源。

**参数：**
- qcheck (默认: 0.5): 唤醒的默认超时，以让重新采样器/重放器知道当前条形图可以检查是否应交付
- historical (默认: False): 如果未提供 `todate` 参数（在基类中定义），如果设置为 True，将强制仅进行历史下载。如果提供了 `todate` 参数，则效果相同。
- milliseconds (默认: True): Visual Chart 构建的条形图具有以下方面：HH:MM:59.999000。如果此参数为 True，将添加一毫秒以使其看起来像：HH:MM + 1:00.000000
- tradename (默认: None): 连续期货不能交易，但非常适合数据跟踪。如果提供此参数，它将是当前期货的名称，即交易资产。例如：

```txt
001ES -> ES-Mini 连续期货作为 dataname 提供
ESU16 -> ES-Mini 2016-09。如果在 tradename 中提供，它将是交易资产。
```

- usetimezones (默认: True): 对于大多数市场，Visual Chart 提供的时区信息允许将日期时间转换为市场时间（backtrader 选择的表示方法）。某些市场（096）需要特殊的内部覆盖和时区支持以显示为用户期望的市场时间。如果设置为 True，将尝试使用 `pytz` 进行时区转换（默认）。禁用它将删除时区使用（可能有助于减少负载）。

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.5)
- calendar (None)
- historical (False)
- millisecond (True)
- tradename (None)
- usetimezones (True)

---

#### VChartCSVData

解析 VisualChart 导出的 CSV 文件。

**特定参数（或特定含义）：**
- dataname: 要解析的文件名或类文件对象

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)

---

#### VChartData

支持 Visual Chart 二进制磁盘文件的每日和日内格式。

**注：**
- dataname: 文件名或打开的类文件对象

如果传递了一个类文件对象，将使用 `timeframe` 参数来确定实际的时间框架。否则将使用文件扩展名（.fd 表示每日，.min 表示日内）。

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)

---

#### VChartFile

支持 Visual Chart 二进制磁盘文件的每日和日内格式。

**注：**
- dataname: Visual Chart 显示的市场代码。例如：015ES 表示 EuroStoxx 50 连续期货

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)

---

#### YahooFinanceCSVData

解析预先下载的 Yahoo CSV 数据源（或符合 Yahoo 格式的本地生成的 CSV 文件）。

**特定参数：**
- dataname: 要解析的文件名或类文件对象
- reverse (默认: False): 假设本地存储的文件在下载过程中已被反向
- adjclose (默认: True): 是否使用股息/拆股调整后的收盘价，并根据它调整所有值
- adjvolume (默认: True): 如果 `adjclose` 也为 True，则也调整交易量
- round (默认: True): 是否在调整收盘价后将值四舍五入到特定的小数位数
- roundvolume (默认: 0): 在调整后将交易量四舍五入到给定的小数位数
- decimals (默认: 2): 要四舍五入的小数位数
- swapcloses (默认: False): [2018-11-16] 看起来关闭和调整后的关闭的顺序现在是固定的。保留该参数，以防需要再次交换列的顺序。

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime
- adjclose

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)
- reverse (False)
- adjclose (True)
- adjvolume (True)
- round (True)
- decimals (2)
- roundvolume (False)
- swapcloses (False

)

---

#### YahooFinanceData

直接从 Yahoo 服务器下载数据。

**特定参数（或特定含义）：**
- dataname: 要下载的代码（例如 'YHOO' 表示 Yahoo 自己的股票报价）
- proxies: 指示下载时使用的代理的字典，例如 {‘http’: ‘http://myproxy.com’} 或 {‘http’: ‘http://127.0.0.1:8080’}
- period: 要下载数据的时间框架。传递 'w' 表示每周，'m' 表示每月。
- reverse: [2018-11-16] Yahoo 在线下载的最新版本返回的数据顺序正确。因此，在线下载的默认值为 False。
- adjclose: 是否使用股息/拆股调整后的收盘价，并根据它调整所有值
- urlhist: Yahoo Finance 中的历史报价 URL，用于获取下载的面包授权 cookie
- urldown: 实际下载服务器的 URL
- retries: 获取面包 cookie 和下载数据的次数

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime
- adjclose

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)
- reverse (False)
- adjclose (True)
- adjvolume (True)
- round (True)
- decimals (2)
- roundvolume (False)
- swapcloses (False)
- proxies ({})
- period (d)
- urlhist ([https://finance.yahoo.com/quote](https://finance.yahoo.com/quote)/{}/history)
- urldown ([https://query1.finance.yahoo.com/v7/finance/download](https://query1.finance.yahoo.com/v7/finance/download))
- retries (3)

---

#### YahooLegacyCSV

此类旨在加载 Yahoo 在 2017 年 5 月停止提供原始服务之前下载的文件。

**数据行：**
- close
- low
- high
- open
- volume
- openinterest
- datetime
- adjclose

**参数：**
- dataname (None)
- name ()
- compression (1)
- timeframe (5)
- fromdate (None)
- todate (None)
- sessionstart (None)
- sessionend (None)
- filters ([])
- tz (None)
- tzinput (None)
- qcheck (0.0)
- calendar (None)
- headers (True)
- separator (,)
- reverse (False)
- adjclose (True)
- adjvolume (True)
- round (True)
- decimals (2)
- roundvolume (False)
- swapcloses (False)
- version ()
