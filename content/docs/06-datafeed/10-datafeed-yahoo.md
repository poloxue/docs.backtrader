---
title: "Yahoo 数据源说明"
weight: 10
---

### Yahoo 数据源说明

在 2017 年 5 月，Yahoo 停用了现有的 CSV 格式的历史数据下载 API。

很快，新 API（这里称为 v7）被标准化并已实现。

这也带来了实际 CSV 下载格式的变化。

#### 使用 v7 API/格式
从版本 1.9.49.116 开始，这是默认行为。可以简单地选择：

- **YahooFinanceData** 用于在线下载
- **YahooFinanceCSVData** 用于离线下载的文件

#### 使用旧的 API/格式
要使用旧的 API/格式，可以：

在线 Yahoo 数据源实例化如下：
```python
data = bt.feeds.YahooFinanceData(
    ...
    version='',
    ...
)
```

离线 Yahoo 数据源实例化如下：
```python
data = bt.feeds.YahooFinanceCSVData(
    ...
    version='',
    ...
)
```

可能在线服务会恢复（服务在没有任何公告的情况下被停用……它也可能会恢复）

或者

仅用于在变更前下载的离线文件，也可以这样做：
```python
data = bt.feeds.YahooLegacyCSV(
    ...
    ...
)
```

新的 **YahooLegacyCSV** 简化了使用 `version=''` 的操作。
