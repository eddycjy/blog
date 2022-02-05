---
title: "Go 读者提问：值为 nil 也能调用函数，太神奇了吧？"
date: 2021-12-31T12:55:27+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

最近在我们 Go 的技术交流群里，有一个小伙伴提了一个程序方面的问题，还挺有意思的，分享给大家。

![](https://files.mdnice.com/user/3610/0774f643-81a2-4458-b557-05c81a9572ef.png)

## 示例

示例程序如下：

```golang
type T struct{}

func (t *T) Hello() string {
	if t == nil {
		fmt.Println("脑子进煎鱼了")
		return ""
	}

	return "煎鱼进脑子了"
}

func main() {
	var t *T
	t.Hello()
```

这段程序的运行结果是什么？

从程序的分析来看，变量 `t` 并没有初始化，只是声明了类型。然后就直接调用了 `Hello` 方法，像是 nil 调用函数，理论上应该出现恐慌（panic）。

运行结果是：

```
panic: runtime error: invalid memory address or nil pointer dereference
```

对不对呢？

显然，真正的运行结果是：

```
脑子进煎鱼了
```

请你思考一下，想想这是为什么？

## 为什么

问题的原因是：很多小伙伴认为变量 `t` 的值都是 nil 了，不应该还能调用到才对。

更抽象化来讲，就是 ”程序是如何检查对象指针来寻找和调度所需函数“。

实际上，在 Go 中，表达式 `Expression.Name` 的语法，所调用的函数完全由 `Expression` 的类型决定。

其调用函数的指向不是由该表达式的特定运行时值来决定，包括我们前面所提到的 nil。

具体如下：

```golang
func (p *Sometype) Somemethod (firstArg int) {}
```

本质上是：

```golang
func SometypeSomemethod(p *Sometype, firstArg int) {}
```

这么一看，其实大家应该都明白了。

上述入参 `p *Sometype` 是有具体上下文类型的，自然而然也就能调用到相应的方法。如果是没有任何上下文类型的，例如：`nil.Somemethod` 方法来调用，那肯定就是无法运行的。

与值是不是 nil，是什么，没有太多直接的影响。只要有预期类型的上下文就可以了。

## 总结

今天给大家分享了一个 Go 语言里面的一个小细节，平时可能很多人没注意到，毕竟 IDE 也会标黄，会避开这个问题点。

在理解 Go 的设计和思考上，我们是需要清晰其背后的原因和逻辑的，也就是类型决定其调用，而不是值（容易误判）。

你有没有遇到过其它的细节问题呢，欢迎交流：）