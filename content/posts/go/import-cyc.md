---
title: "为什么 Go 不支持循环引用？"
date: 2021-12-31T12:55:15+08:00
toc: true
images:
tags: 
  - go
  - 为什么
---

大家好，我是煎鱼。

近年来开始学习 Go 语言的开发者越来越多了。很多小伙伴在使用时，就会遇到种种不理解的问题。

其中一点就是循环引入的报错：

```shell
package command-line-arguments
	imports github.com/eddycjy/awesome-project/a
	imports github.com/eddycjy/awesome-project/b
	imports github.com/eddycjy/awesome-project/a: import cycle not allowed
```

为什么 Go 不支持循环引用呢，这就很不解了，难道还影响性能了？


![图来自网络](https://files.mdnice.com/user/3610/4596a6d1-9286-40b2-8074-cc43f9bbd9f4.png)

今天煎鱼将和大家一起了解背后的原因。

## 案例演示

做一个基本的案例 Demo，便于没接触过的同学建立初步认知。我们的程序分别有 2 个 package。

package a 的代码如下：

```golang
import (
	"github.com/eddycjy/awesome-project/b"
)

func Hello(s string) {
	b.Print(s)
}
```

package b 的代码如下：

```golang
import (
	"fmt"

	"github.com/eddycjy/awesome-project/a"
)

func Hello() {
	a.Hello("脑子进煎鱼了")
}

func Print(s string) {
	fmt.Println(s)
}
```

再在 main.go 的文件中调用 `a.Hello("脑子进煎鱼了")` 方法。

一运行，就会出现如下错误提示：

```shell
package command-line-arguments
	imports github.com/eddycjy/awesome-project/a
	imports github.com/eddycjy/awesome-project/b
	imports github.com/eddycjy/awesome-project/a: import cycle not allowed
```

错误的本质原因是 package a 引用了 package b，而 package b 又引用了 package a，造成了循环引用。

这在 Go 语言中是明令禁止的，在编译时就会中断程序，导致编译失败。

## 原因分析

根据现在 Go 官方的统一意见来看，package 循环导入几乎不可能出现，即使是 Go2。

因为 Go2 可能是很多核心问题的破变的关键节点，有许多人提了类似《[proposal: Go 2: allow import cycle](https://github.com/golang/go/issues/30247)》的提案，希望解决循环引入的问题。

Go 语言之父 Rob Pike 亲自回答了这个问题，原因如下：

- 没有支持循环导入，目的是迫使 Go 程序员更多地考虑程序的依赖关系。
    - 保持依赖关系图的简洁。
    - 快速的程序构建。
- 如果支持循环导入，很容易会造成懒惰、不良的依赖性管理和缓慢的构建。这是设计者不希望看见的。
    - 混乱的依赖关系。
    - 缓慢的程序构建

简单拿来就，就是在 Go 工程中出现循环引用，这会对构建性能和依赖关系的解决非常不利。

为此，考虑一开始就保持图的正确 DAG，认为这是一个值得预先简化的领域。导入循环可能很方便，但其实背后的代价可能是灾难性的，所以在 Go 中被明确禁止支持。

## 总结

在程序中，如果我们频繁的出现模块与模块之间的循环引用，这时候我们是不是应该考虑一下，是不是设计的有些问题，要不要考虑调整？

但也并非所有的事都是二极管，Go 源码可能或多或少都有自己循环引用的案例，最重要的是想清楚。

**你对此支持循环引用怎么看**，欢迎在评论区留言交流：）
