---
title: "快报：正式提案将泛型特性加入 Go 语言"
date: 2021-01-13T21:11:44+08:00
toc: true
images:
tags: 
  - 泛型
---

经历九九八十一难，多年的不断探讨和 Go 语言爱好者们在社区中的强烈关注，且 Go 官方在 2020 年不断放出消息。

![image](https://image.eddycjy.com/1d0e5a264c65e37659f142bc2ee55805.jpg)

总算在 2021 年 1 月 12 日。官方正式提出将泛型特性加入 Go 语言的 proposal 了，且最新的草案设计已经更新。


基本语法如下：

```
func Print[T any](s []T) {
	// same as above
}
```

其大体的概述如下：

- 函数可以具有使用方括号的其他类型参数列表，但其他情况下看起来像普通的参数列表：`func F[T any](p T) { ... }`。
- 类型也可以具有类型参数列表：`type MySlice[T any] []T`。
- 每个类型参数都有一个类型约束，就像每个普通参数都有一个类型：`func F[T Constraint](p T) { ... }`。
- 类型约束是接口类型。
- 新的预声明名称 `any` 是允许任何类型的类型约束。
- 用作类型约束的接口类型可以具有预先声明的类型的列表。只有与那些类型之一匹配的类型参数才能满足约束条件。
- 泛型函数只能使用其类型约束所允许的操作。
- 使用泛型函数或类型需要传递类型实参。
- 在通常情况下，类型推断允许省略函数调用的类型参数。

根据官方博客的消息，如果该提案被正式接受。那么将会在 2021 年底之前完成一个基本可用的泛型特性使用，又或是会作为 Go1.18beta 的一部分。

这是 Go 泛型特性的又一步前进。若大家有兴趣进一步了解或想提出意见，可查看下述传送门：

- A Proposal for Adding Generics to Go：https://blog.golang.org/generics-proposal。
- proposal: spec: add generic programming using type parameters：https://github.com/golang/go/issues/43651。

今年年底或 Go1.18beta 到底能不能看到泛型的正式完整可用版本呢，值得期待。