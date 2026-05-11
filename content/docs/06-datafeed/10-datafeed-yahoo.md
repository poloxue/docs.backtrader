---
title: "Yahoo 数据源说明"
weight: 10
---

# Yahoo 数据源说明

2017 年 5 月，Yahoo 停用了原有的 CSV 格式历史数据下载 API。随后新的 API（这里称为 v7）被标准化并已实现，也带来了 CSV 下载格式的变化。

## 使用 v7 API/格式
从 1.9.49.116 版本开始，这是默认行为。直接使用：

- **YahooFinanceData** 用于在线下载
- **YahooFinanceCSVData** 用于离线文件

## 使用旧的 API/格式
使用旧的 API/格式时：
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

在线服务可能会恢复（服务在没有任何公告的情况下被停用，也可能会恢复）。

或者，针对变更前下载的离线文件，也可以这样做：
```python
data = bt.feeds.YahooLegacyCSV(
    ...
    ...
)
```

新的 **YahooLegacyCSV** 简化了 `version=''` 的用法。
