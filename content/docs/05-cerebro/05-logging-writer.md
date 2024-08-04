---
title: "日志 Logging - Writter"
weight: 5
---

### Writer

Writer类负责将以下内容写入流：

- 包含数据源、策略、指标和观察者的CSV流。

可以通过每个对象的`csv`属性控制哪些对象实际进入CSV流（数据源和观察者默认为True，指标默认为False）。

以下属性的摘要：

- 数据源
- 策略（线条和参数）
- 指标/观察者（线条和参数）
- 分析器（参数和分析结果）

系统中定义了一个名为`WriterFile`的Writer，可以通过以下方式添加：

1. 设置Cerebro的`writer`参数为True
   - 将实例化一个标准的`WriterFile`
2. 调用`Cerebro.addwriter(writerclass, **kwargs)`
   - 在回测执行期间，使用给定的kwargs实例化`writerclass`

由于标准的`WriterFile`默认不输出CSV，以下调用可以处理这一点：

```python
cerebro.addwriter(bt.WriterFile, csv=True)
```

### 参考

#### `class backtrader.WriterFile()`

系统范围内的Writer类。

可以通过以下参数进行参数化：

- `out`（默认：`sys.stdout`）：写入的输出流
  - 如果传递的是字符串，将使用参数内容作为文件名。
- `close_out`（默认：False）
  - 如果`out`是一个流，是否需要由Writer显式关闭。
- `csv`（默认：False）
  - 在执行过程中，是否将数据源、策略、观察者和指标的CSV流写入输出流。
  - 可以通过每个对象的`csv`属性控制哪些对象实际进入CSV流（数据源和观察者默认为True，指标默认为False）。
- `csv_filternan`（默认：True）
  - 是否需要从CSV流中清除nan值（用空字段替换）。
- `csv_counter`（默认：True）
  - Writer是否应保持并输出实际输出行的计数器。
- `indent`（默认：2）
  - 每个级别的缩进空格数。
- `separators`（默认：['=', '-', '+', '*', '.', '~', '"', '^', '#']）
  - 用于分隔部分/子部分的行分隔符字符。
- `seplen`（默认：79）
  - 包括缩进在内的行分隔符的总长度。
- `rounding`（默认：None）
  - 将浮点数舍入到的小数位数。如果为None，则不执行舍入。
