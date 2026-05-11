---
title: "日志 Logging - Writter"
weight: 6
---

# Writer

Writer 类负责将以下内容写入输出流：

数据源、策略、指标和观察者的 CSV 流。可通过每个对象的 `csv` 属性控制哪些对象输出到 CSV 流（数据源和观察者默认为 True，指标默认为 False）。

属性摘要：

- 数据源
- 策略（线条和参数）
- 指标/观察者（线条和参数）
- 分析器（参数和分析结果）

系统中定义了一个名为 `WriterFile` 的 Writer，可通过以下方式添加：

1. 将 Cerebro 的 `writer` 参数设为 True，会实例化一个标准的 `WriterFile`
2. 调用 `Cerebro.addwriter(writerclass, **kwargs)`，在回测执行期间使用给定的 kwargs 实例化 `writerclass`

由于标准 `WriterFile` 默认不输出 CSV，可使用如下方式启用：

```python
cerebro.addwriter(bt.WriterFile, csv=True)
```

## 参考

### `class backtrader.WriterFile()`

系统范围内的 Writer 类。

可以通过以下参数进行参数化：

参数名          | 默认         | 说明
--------------- | ------------ | ------------------
`out`           | `sys.stdout` | 写入的输出流。如果传递的是字符串，则将其作为文件名。
`close_out`     | `False`      | 如果 `out` 是流对象，是否需要由 Writer 显式关闭。
`csv`           | `False`      | 是否将数据源、策略、观察者和指标的 CSV 流写入输出。可通过每个对象的 `csv` 属性控制哪些对象输出到 CSV（数据源和观察者默认为 True，指标默认为 False）。
`csv_filternan` | `True`       | 在 CSV 输出中清除 NaN 值（替换为空字段）。
`csv_counter`   | `True`       | 是否记录并输出实际输出行的计数器。
`indent`        | 2            | 每个级别的缩进空格数。
`separators`    | `'=', '-', '+', '*', '.', '~', '"', '^', '#'` | 用于分隔部分/子部分的行分隔符字符。
`seplen`        | 79           | 包括缩进在内的行分隔符的总长度。
`rounding`      | None         | 浮点数保留的小数位数。如果为 None，则不做舍入。
