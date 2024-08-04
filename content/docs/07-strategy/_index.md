---
title: "策略类 Strategy"
weight: 7
bookCollapseSection: true
---

## 数据绑定

数据绑定是 Fyne 工具包的一个强大新功能，它在版本 `v2.0.0` 中引入。通过使用数据绑定，我们可以避免手动管理许多标准对象，如 `Label`、`Button` 和 `List`。

内置绑定支持许多原始类型（如 `Int`、`String`、`Float` 等）、列表（例如 `StringList`、`BoolList`）以及 `Map` 和 `Struct` 绑定。每种类型都可以使用简单的构造函数创建。例如，要使用零值创建一个新的字符串绑定，可以使用 `binding.NewString()`。你可以使用 `Get` 和 `Set` 方法获取或设置大多数数据绑定的值。

也可以使用以 `Bind` 开头的类似函数绑定到现有值，它们都接受指向绑定类型的指针。要绑定到现有的 `int`，我们可以使用 `binding.BindInt(&myInt)`。通过保留对绑定值的引用而不是原始变量，我们可以配置控件和函数以自动响应任何更改。如果你直接更改外部数据，请确保调用 `Reload()` 以确保绑定系统读取新值。

```go
package main

import (
	"log"

	"fyne.io/fyne/v2/data/binding"
)

func main() {
	boundString := binding.NewString()
	s, _ := boundString.Get()
	log.Printf("Bound = '%s'", s)

	myInt := 5
	boundInt := binding.BindInt(&myInt)
	i, _ := boundInt.Get()
	log.Printf("Source = %d, bound = %d", myInt, i)
}
```

接下来我们开始学习关于[简单绑定](/docs/07-binding/01-simple)控件绑定。
