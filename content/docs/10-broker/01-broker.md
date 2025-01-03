---
title: "Broker"
weight: 1
---

# Broker 

## 类 `backtrader.brokers.BackBroker()`

Broker 经纪商模拟器，该模拟支持不同的订单类型，检查提交订单的现金需求与当前现金的对比，跟踪每次 Cerebro 迭代的现金和价值，并保持不同数据的当前头寸。

现金在每次迭代中调整，对于期货等工具来说，当价格变化时，会在实际经纪商中增加或减少现金。
支持的订单类型：

- `Market`：将在下一个柱的第一个tick（即开盘价）执行
- `Close`：用于日内交易，订单以会话最后一个柱的收盘价执行
- `Limit`：如果在会话期间看到给定的限价则执行
- `Stop`：如果看到给定的止损价，则执行市场订单
- `StopLimit`：如果看到给定的止损价，则启动限价订单

因为经纪商由 Cerebro 实例化，用户通常不需要替换经纪商实例，因此参数不受用户控制。

要更改参数，有两种选择：

1. 手动创建带所需参数的 Broker，用 `cerebro.broker = instance` 将该实例设置为这个经纪商；
2. 使用 `set_xxx` 方法通过 `cerebro.broker.set_xxx` 设置参数，其中 `xxx` 代表设置参数名称；

**注意**，`cerebro.broker` 是一个由 Cerebro 的 `getbroker` 和 `setbroker` 方法支持的属性。

## 参数

参数名         | 默认值                         | 描述
-------------- | ------------------------------ | -------------
`cash`         | 10000                          | 起始现金
`commission`   | `CommInfoBase(percabs=True)`   | 适用于所有资产的基础佣金方案
`checksubmit`  | True                           | 在将订单接受到系统之前检查保证金/现金
`eosbar`       | False                          | 对于日内 Bar，考虑与会话结束时间相同的柱为会话结束。通常不会这样，因为许多交易所会在会话结束后几分钟内为许多产品生成一些柱（最终拍卖）
`filler`       | None                           | 一个可调用对象，签名为 `callable(order, price, ago)`。<br/><br/>参数说明：<ul style="list-style-type: none;padding-left: 0; margin-left: 0;"><li>- `order`：显然是执行中的订单。这提供了对数据的访问（包括 ohlc 和成交量值）、执行类型、剩余大小（`order.executed.remsize`）等。</li><li>- `price`：订单将在 `ago` 柱中执行的价格</li><li>- `ago`：用于从 `order.data` 提取 ohlc 和成交量价格的索引。在大多数情况下，这将是 0，但在某些角落情况下，对于 `Close` 订单，这将是 -1。</li><li>- 可调用对象必须返回执行的大小（值 >= 0）</li></ul>可调用对象当然可以是一个 `__call__` 符合上述签名的对象。默认情况下，订单将一次性完全执行。
`slip_perc`    | 0.0                            | 用于买卖订单上下滑动价格的绝对百分比（且为正值）。<br/>注意：0.01 是 1%，0.001 是 0.1%
`slip_fixed`   | 0.0                            | 用于买卖订单上下滑动价格的单位百分比（且为正值）。<br/>注意：如果 `slip_perc` 非零，则优先于此。
`slip_open`    | False                          | 是否滑动专门使用下一个柱的开盘价执行的订单价格。例如，市场订单将在下一个可用tick执行，即柱的开盘价。这也适用于其他一些执行，因为逻辑尝试检测开盘价是否会匹配请求的价格/执行类型在移动到新柱时。
`slip_match`   | True                           | - 如果为 True，经纪商将通过在高/低价位封顶滑点来提供匹配，以防它们超出。<br/><br/>- 如果为 False，经纪商将不会使用当前价格匹配订单，并将在下一次迭代中尝试执行
`slip_limit`   | True                           | - 限价订单，给定确切的匹配价格请求，即使 `slip_match` 为 False，也会被匹配。<br/>- 此选项控制该行为。<br/>- 如果为 True，那么限价订单将通过在限价/高低价位封顶价格进行匹配<br/>- 如果为 False 且滑点超出上限，则不会有匹配<br/>
`slip_out`     | False                          | 即使价格超出高-低范围，也提供滑点。
`coc`          | False                          | Cheat-On-Close 将其设置为 True 与 `set_coc` 启用，将“市场”订单与订单条的收盘价匹配。这实际上是作弊，因为柱已关闭，任何订单都应首先与下一个柱的价格匹配
`coo`          | False                          | Cheat-On-Open 将其设置为 True 与 `set_coo` 启用，将“市场”订单与开盘价匹配，例如使用设置为 True 的计时器，因为这种计时器在经纪商评估之前执行
`int2pnl`      | True                           | 将生成的利息（如果有）分配给减少头寸的操作的利润和亏损（无论是多头还是空头）。在某些情况下，这可能是不希望的，因为不同的策略在竞争，利息将以不确定的方式分配给其中任何一个。
`shortcash`    | True                           | 如果为 True，则在卖空类似股票的资产时将增加现金，并且该资产的计算价值将为负值。<br/>- 如果为 False，则现金将作为操作成本扣除，计算的价值将为正值，以最终得到相同的金额
`fundstartval` | 100.0                          | 此参数控制基金式绩效测量的起始值，即：现金可以增加和扣除，增加股票数量。绩效不是使用投资组合的净资产价值来衡量，而是使用基金的价值
`fundmode`     | False                          | 如果设置为 True，诸如 TimeReturn 的分析器可以基于基金价值而不是总净资产价值自动计算回报

## 方法

签名                            | 描述
------------------------------- | ------------------------
set_cash(cash)                  | 设置现金参数（别名：setcash）
get_cash()                      | 返回当前现金（别名：getcash）
get_value(<br/>&emsp;datas=None, <br/>&emsp;mkt=False, <br/>&emsp;lever=False<br/>) | 返回给定数据的投资组合价值（如果数据为 None，则返回总投资组合价值）（别名：getvalue）
set_eosbar(eosbar)              | 设置 eosbar 参数（别名：seteosbar）
set_checksubmit(checksubmit)    | 设置 checksubmit 参数
set_filler(filler)              | 设置用于成交量填充执行的填充器
set_coc(coc)                    | 配置 Cheat-On-Close 方法以在订单柱上买入收盘价
set_coo(coo)                    | 配置 Cheat-On-Open 方法以在订单柱上买入收盘价
set_int2pnl(int2pnl)            | 配置将利息分配给利润和亏损
set_fundstartval(fundstartval)  | 设置基金式绩效跟踪器的起始值
set_slippage_perc(<br/>&emsp;perc, <br/>&emsp;slip_open=True, <br/>&emsp;slip_limit=True, <br/>&emsp;slip_match=True, <br/>&emspslip_out=False<br/>) | 配置滑点为基于百分比
set_slippage_fixed(<br/>&emsp;fixed, <br/>&emsp;slip_open=True, <br/>&emsp;slip_limit=True, <br/>&emsp;slip_match=True, <br/>&emsp;slip_out=False<br/>) | 配置滑点为固定点数
get_orders_open(safe=False)     | 返回仍然打开的订单（未执行或部分执行）的可迭代对象。返回的订单不得被触摸。如果需要订单操作，请将参数 safe 设置为 True
getcommissioninfo(data)         | 检索与给定数据相关的 `CommissionInfo` 方案
setcommission(<br/>&emsp;commission=0.0, <br/>&emsp;margin=None, <br/>&emsp;mult=1.0, <br/>&emsp;commtype=None, <br/>&emsp;percabs=True, <br/>&emsp;stocklike=False, <br/>&emsp;interest=0.0, <br/>&emsp;interest_long=False, <br>&emsp;leverage=1.0, <br/>&emsp;automargin=False, <br/>&emsp;name=None<br/>)  | 为 Broker 设置 `CommissionInfo` 对象，参考 `CommInfoBase` 文档介绍。<br/><br/>如果 name 为 None，这将是没有找到其他 `CommissionInfo` 方案的资产的默认设置
addcommissioninfo(<br/>&emsp;comminfo, name=None<br/>) | 添加 `CommissionInfo` 对象，如果 name 为 None，将成为所有资产的默认设置
getposition(data)               | 返回给定数据的当前头寸状态（一个 `Position` 实例）
get_fundshares()                | 返回基金模式中的当前股票数量
get_fundvalue()                 | 返回基金式的股票价值
add_cash(cash)                  | 添加/移除系统现金（使用负值移除）

