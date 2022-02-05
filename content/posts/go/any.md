---
title: "Go 新关键字 any，interface 会成历史吗？"
date: 2021-12-31T12:55:21+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

大家在看 Go1.18 泛型的代码时，不知道是否有留意到一个新的关键字 any。

例子如下：

```golang
func Print[T any](s []T) {}
```

之前没有专门提过，但有没有小伙伴以为这个关键字，是泛型代码专属的？

其实不是...在这次新的 Go1.18 更新中，any 是作为一个新的关键字出现，**any 有一个真身，本质上是 interface{} 的别名**：

```golang
type any = interface{}
```

也就是，在常规代码中，也可以直接使用：

```golang
func f(a any) {
	switch a.(type) {
	case int:
		fmt.Println("进脑子煎鱼了")
	case float64:
		fmt.Println("煎鱼进脑子了")
	case string:
		fmt.Println("脑子进煎鱼了")
	}
}

func main() {
	f(2)
	f(3.1415)
	f("煎鱼好！")
}
```

从使用层面来讲，新的关键字 any 会比 interface{} 方便不少，毕竟少打了好多个词，更快了，其实也是参照现有 rune 类型的做法。

增加新关键字后的对比如下：

| 长声明    |  短声明   |
| --- | --- |
|  func f[T interface{}](s []T) []T	   |  func f[T any](s []T) []T
  func f(a interface{})	 | func f(a any)
 |  var a interface{}	   |  var a any   |

我们在了解他的便利性后，再从代码一致性和可读性来讲，是有些问题的，会造成一定的疑惑。

因此前两天有人提出了《[all: rewrite interface{} to any](https://github.com/golang/go/issues/49884)》的需求，打算把内部所有的代码都重写一遍。

![](https://files.mdnice.com/user/3610/06687ddd-e224-4499-892b-3869ee1fc1d0.png)

你可能会以为是人肉手工改？那肯定不是，Go 官方发起了 CL 进行批量修改。

我们在日常的工程中，也可以和他们一样，直接借用 Go 工具链来实现替换。

如下：

```golang
gofmt -w -r 'interface{} -> any' ./...
```

听到这个消息时，我的朋友咸鱼就大惊了，在想 interface{} 会不会成为历史，被新的关键字 any 完全替代？

![](https://files.mdnice.com/user/3610/41a70c0c-284c-44c7-9b15-a1ee37545424.png)

显然，答案是不会的。因为 **Go1 有兼容性的保证**，肯定不会在现阶段删除。不过后续会出现一个现象，就是我们的 Go 工程中，有人用 any，有人用 interface{}，会在代码可读性上比较伤人。

不过我们也可以学 Go 官方，在 linter 流程中借助 gofmt 工具来强行把所有 interface{} 都替换成 any 来实现代码的一致性。 

这次变更，感觉是个**美学**的问题，你对此怎么看呢？有没有也希望哪些东西有别名，**欢迎大家在评论区留言和交流**：）