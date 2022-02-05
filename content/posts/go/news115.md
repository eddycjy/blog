---
title: "分享 Go 最近的几件周边大小事"
date: 2021-12-31T12:55:17+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

最近可能是因为 Q4 了，又恰逢 Go1.18 快要发布，各路 Go 语言的新消息层出不穷。

今天煎鱼带大家一起来了解下最近社区发生的几件大小事，当然，我只讲一些核心的内容。

![](https://files.mdnice.com/user/3610/d6f8bfde-141c-405b-8ec5-779d31b0283d.png)

## Go 诞生 12 年

在 2021 年 11 月 10 日，是 Go 语言开源版本的 12 岁生日。Go 官方博客发表《[Twelve Years of Go](https://go.dev/blog/12years "Twelve Years of Go")》。

主体内容分为三块介绍：
- 回顾过去一年的核心变更。
- 展望明年的特性计划。
- 介绍今年做的 Go 相关分享。

### 回顾过去

是对过去一年的版本更新进行说明：
- Go1.16：默认启用 Go modules，增加 MacOS ARM64 的支持，新支持文件系统接口和嵌入文件的特性。
- Go1.17：Go 函数改为基于寄存器的调用规范（提高了 5~15% 的性能），增加 Windows ARM64 支持，还引入了模块裁剪等功能。
- Go1.18：预计支持模糊测试（Go fuzzing）、泛型等强大的新特性。

核心总结：今年大力推动 Go modules，提高了 Go 函数的性能，增加了更多的计算机架构支持，以及若干改进和优化（例如：TLS）。

也为了推广 Go 语言，做出了更多的努力和资料培训（Gin 也因此更上了一层楼）。

### 后续安排

我们核心关注泛型方面的消息，泛型将会是 2022 年的核心重点之一。规划如下：
- Go 1.18 中的初始版本只是泛型的开始，将会在此版本使用泛型并学习哪些有效、哪些无效。
- 在确定泛型的 “实践” 后，会输出 “最佳实践”，并决定何时追加泛型实现到标准库和第三方库中。
- 期待在 **Go1.19**（也就是 2022.08）及更高版本将**进一步完善泛型**的设计和实现，并将它们进一步整合到整体的 Go 体验中（也就是工具链等）。

核心总结：明年要继续大力推进泛型，先尝鲜，再出最佳实践，进而融合进 Go 体系中，路还比较远。

## 分享知识

今年 Go 团队为了推广 Go 语言的配套知识体系，还发布了一堆教程：
- 《[guide to databases in Go](https://golang.org/doc/database/ "guide to databases in Go")》：Go 数据库指南。
- 《[guide to developing modules](https://golang.org/doc/#developing-modules "guide to developing modules")》：Go 开发模块指南。
- 《[用 Go 和 Gin 开发 RESTful API](https://golang.org/doc/tutorial/web-service-gin "用 Go 和 Gin 开发 RESTful API")》：用 Go 和 Gin 开发 RESTful API，有种把 Gin 作为推荐框架的感觉了。

这两天 Go 团队在 Google Open Source Live 举办了第二次年度 Go 日：

![来源于 youtube，还自带字幕翻译](https://files.mdnice.com/user/3610/ba469b39-4395-4f3c-9e82-37e4a1b8812e.png)  

分享了如下话题：
- 《[Using Generics in Go](https://www.youtube.com/watch?v=nr8EpUO9jhw "Using Generics in Go")》：介绍了泛型以及如何有效地使用它们。
- 《[Modern Enterprise Applications](https://www.youtube.com/watch?v=5fgG1qZaV4w "Modern Enterprise Applications")》：展示了 Go 在企业现代化中所扮演的角色。
- 《[Building Better Projects with the Go Editor](https://www.youtube.com/watch?v=jMyzsp2E_0U "Building Better Projects with the Go Editor")》：展示了VS Code Go的集成工具如何帮助你浏览代码、调试测试等。
- 《[From Proof of Concept to Production](https://www.youtube.com/watch?v=e7PtBOsTpXE "From Proof of Concept to Production")》：介绍美国运通公司如何在其支付和奖励平台中使用 Go。

有个别 Go 话题还是很不错的，尤其是泛型的快速了解。这在该视频网站上还有比较顺畅的字幕翻译，阅读基本没问题。

可自行选择食用。

## 泛型新语法和 Playground

### GoTip Playground

在之前我们是通过 `go2goplay.golang.org` 来进行一些在线泛型的例子玩耍。

在经历了一定时间的迭代后，泛型的新特性多了不少，由新版的 Playground（gotipplay.golang.org）来替换使用：

![https://gotipplay.golang.org](https://files.mdnice.com/user/3610/53d4f09e-689b-445f-862c-f18b91a359f8.png)

主要特点：所运行的代码**基于 tip 版本**，再也不用自己拉代码了。在 select 控件上也多了不少例子的展示。

后续大概率会增加泛型的实践。

### 约束语法

需要注意的一点，最新的 Go 泛型约束语法又又又变了。从原本的 “,” 改为了 "|"。

![Type Parameters Proposal](https://files.mdnice.com/user/3610/5f1f20c6-0d81-43a1-a1c3-334213546fc7.png)

原本如下：

```golang
type T interface {
 type int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64, uintptr, float32, float64, complex64, complex128, string
}
```

新的如下：

```golang
type T interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64 |
		~string
}
```

## 新搜索体验

近期官方对文档站 `pkg.go.dev` 做了一轮搜索优化。主要分为如下：
- 按包搜索。
- 按符号搜索。

若是按包搜索，将会对相关包的搜索结果进行分组，优化后不再是流水式展示，而是聚类，根据归类后分组展示。

如下图：

![按包搜索](https://files.mdnice.com/user/3610/5e12b6ee-995a-4dd4-af4d-8aec6f57f580.png)

若是按符号搜索，将可以对包内的 “符号” 实现更精准的搜索，例如：常量、变量、函数、类型、字段或方法。

如下图：

![按符号搜索](https://files.mdnice.com/user/3610/52697a9c-00f1-46d8-9068-976d85dcb6d0.png)

搜索效率是提高了不少的。以后会不会向完整的代码搜索和关联的方向发展，也是个值得思考的问题。

## 总结

今天给大家介绍了 Go 语言社区最近发生的好几件大小事，每一个展开都是一篇新的文章。
例如：可以了解为什么泛型要从 “,” 改为 “|” 的格式，肯定是有原因的。

这些内容（以及 Go1.18 的新特性），我会在后续的文章继续深入展开和介绍。

欢迎大家关注我，获取一手的 Go 社区快讯和知识：）