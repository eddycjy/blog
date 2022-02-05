---
title: "小技巧分享：在 Go 如何实现枚举？"
date: 2021-12-31T12:54:50+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

在日常的业务工程开发中，我们常常会有使用枚举值的诉求，枚举控的好，测试值边界一遍过...

有的小伙伴会说，在 Go 语言不是有 `iota` 类型做枚举吗，那煎鱼你这篇文章还讲什么？

讲道理，Go 语言并没有 enum 关键字，有用过 Protobuf 等的小伙伴知道，Go 语言只是 ”有限的枚举“ 支持，我们也会用常量来定义，枚举值也需要有字面意思的映射。

示例
--

在一些业务场景下，是没法达到我们的诉求的。示例如下：

```
type FishType int

const (
 A FishType = iota
 B
 C
 D
)

func main() {
 fmt.Println(A, B, C, D)
}
```

输出结果为：“0 1 2 3”。这时候就一脸懵逼了...枚举值，应该除了键以外，还得有个对应的值。也就是这个 “0 1 2 3” 分别对应着什么含义，是不是应该输出 ”A B C D“

但 Go 语言这块就没有直接的支撑了，因此这不是一个完整的枚举类型的实现。

同时假设我们传入超过 `FishType` 类型声明范围的枚举值，在 Go 语言中默认也不会有任何控制，是正常输出的。

上述这种 Go 枚举实现，在某种情况下是不完全的，严格意义上不能成为 enum（枚举）。

使用 String 做枚举
-------------

如果要支持枚举值的对应输出的话，我们可以通过如下方式：

```
type FishType int

const (
 A FishType = iota
 B
 C
 D
)

func (f FishType) String() string {
 return [...]string{"A", "B", "C", "D"}[f]
}
```

运行程序：

```
func main() {
 var f FishType = A
 fmt.Println(f)
 switch f {
 case A:
  fmt.Println("脑子进煎鱼了")
 case B:
  fmt.Println("记得点赞")
 default:
  fmt.Println("别别别...")
 }
}
```

输出结果：

```
A
脑子进煎鱼了
```

我们可以借助 Go 中 `String` 方法的默认约定，针对于定义了 `String` 方法的类型，默认输出的时候会调用该方法。

这样就可以达到获得枚举值的同时，也能拿到其映射的字面意思。

自动生成 String
-----------

但每次手动编写还是比较麻烦的。在这一块，我们可以利用官方提供的 `cmd/string` 来快速实现。

我们安装如下命令：

```
go install golang.org/x/tools/cmd/stringer
```

在所需枚举值上设置 `go:generate` 指令：

```
//go:generate stringer -type=FishType
type FishType int
```

在项目根目录执行：

```
go generate ./...

```

会在根目录生成 fishtype\_string.go 文件：

```
.
├── fishtype_string.go
├── go.mod
├── go.sum
└── main.go
```

fishtype\_string 文件内容：

```
package main

import "strconv"

const _FishType_name = "ABCD"

var _FishType_index = [...]uint8{0, 1, 2, 3, 4}

func (i FishType) String() string {
 if i < 0 || i >= FishType(len(_FishType_index)-1) {
  return "FishType(" + strconv.FormatInt(int64(i), 10) + ")"
 }
 return _FishType_name[_FishType_index[i]:_FishType_index[i+1]]
}
```

所生成出来的文件，主要是根据枚举值和映射值做了个映射，且针对超出枚举值的场景进行了判断：

```
func main() {
 var f1 FishType = A
 fmt.Println(f1)
 var f2 FishType = E
 fmt.Println(f2)
}
```

执行 `go run .` 查看程序运行结果：

```
$ go run .
A
FishType(4)
```

总结
--

在今天这篇文章中，我们介绍了如何在 Go 语言实现标准的枚举值，虽然有些繁琐，但整体不会太难。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05d5614987b04e0183e20fb5ac5249d4~tplv-k3u1fbpfcp-zoom-1.image)

也有小伙伴已经在社区中提出了 ”proposal: spec: add typed enum support“ 的提案，相信未来有机会能看到 Go 自身支持 enum（枚举）的那一天。

你平时会怎么在业务代码中实现枚举呢，欢迎大家一起留言交流：）


**欢迎大家在评论区留言和交流 ：）**

## 鼓励

若有任何疑问欢迎评论区反馈和交流，**最好的关系是互相成就**，各位的**点赞**就是[煎鱼](https://github.com/eddycjy)创作的最大动力，感谢支持。

> 文章持续更新，可以微信搜【脑子进煎鱼了】阅读，本文 **GitHub** [github.com/eddycjy/blog](https://github.com/eddycjy/blog) 已收录，学习 Go 语言可以看 [Go 学习地图和路线](https://github.com/eddycjy/go-developer-roadmap)，欢迎 Star 催更。