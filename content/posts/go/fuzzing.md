---
title: "提高 Go 程序健壮性，Fuzzing 要来了！"
date: 2021-12-31T12:54:50+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

就在前几天，Go1.17 beta1 正式发布：

![](https://files.mdnice.com/user/3610/fb4ceb3d-ef1c-4c32-a7c3-a4188a0d74b0.png)

兴冲冲本想着看一下当初在 Go1.17 的计划中，预计会支持的新特性：模糊测试（Fuzzing）。不过没想到...计划赶不上变化，官方正式宣告 Fuzzing 不会出现在 Go1.17 的新功能中。

煎鱼在悲伤之际，发现 Go 在 dev.fuzz 分支上提供了该功能的 Beta 测试，因此今天带大家一起来深入该特性。

## 什么是 Fuzzing

Fuzzing 是一种自动测试技术，包括向计算机程序提供随机数据作为输入。然后监测程序是否出现恐慌、断言失败、无限循环等。

Fuzzing 不是使用一个小的、预先定义好的手动创建的输入集（如单元测试），而是用新的案例不断地测试代码，以努力 ”锻炼“ 有关软件的所有方面。

这听起来很 ”难“。但在过去的几年里，Fuzzing 的技术水平有了很大的提高。Fuzzing 不再是需要专业知识才能成功使用的东西，现代模糊测试策略能更快、更有效地找到有用的输入。

在应用程序中，就是你只要引入一个 package，对着 API 一顿用就可以了。

## 为什么要做 Fuzzing

可能会有小伙伴说，测试？直接人工测试，再把测试数据准备一下，配套 YAPI 等接口管理平台，把自动化接口测试一弄就好了。还需要 Fuzzing 吗？

其实 Fuzzing 是对其他形式的测试、代码审查和静态分析的补充，它通过生成一个随机测试用例去覆盖人为测不到的各种复杂场景。而这些输入几乎不可能人为去构造，总会被传统测试所遗漏。

## 发生在身边的 Fuzzing

实际上 Go-fuzz 对 Go 标准库进行过测试，依然这这之中发现了 200  多个 bug：

![](https://files.mdnice.com/user/3610/9b00972c-2231-4406-84b8-2e8e44f57d6d.png)

这还是建立在标准库已经比较成熟，且由非常有经验的开发者编写，在生产中使用多年的情况下，依然有如此多的问题。

## 快速上手

我们需要在本地执行如下命令，需开启 GO111MODULE 和天梯：

```golang
$ go get golang.org/dl/gotip
$ gotip download dev.fuzz
```

执行完毕后会从 dev.fuzz 分支构建 Go 工具链，同时 gotip 可以作为 go 命令的替代者命令，也就是可以运行 Fuzzing 的相关代码了。

```golang
// +build gofuzzbeta

package tests

import (
	"net/url"
	"reflect"
	"testing"
)

func FuzzParseQuery(f *testing.F) {
	f.Add("x=1&y=2")
	f.Fuzz(func(t *testing.T, queryStr string) {
		query, err := url.ParseQuery(queryStr)
		if err != nil {
			t.Skip()
		}
		queryStr2 := query.Encode()
		query2, err := url.ParseQuery(queryStr2)
		if err != nil {
			t.Fatalf("ParseQuery failed to decode a valid encoded query %s: %v", queryStr2, err)
		}
		if !reflect.DeepEqual(query, query2) {
			t.Errorf("ParseQuery gave different query after being encoded\nbefore: %v\nafter: %v", query, query2)
		}
	})
}
```

在相应的目录下执行 `gotip test -fuzz=FuzzParseQuery` 命令，输出结果：

```golang
fuzzing, elapsed: 3.0s, execs: 319 (106/sec), workers: 4, interesting: 15
fuzzing, elapsed: 6.0s, execs: 665 (111/sec), workers: 4, interesting: 15
fuzzing, elapsed: 9.0s, execs: 1019 (113/sec), workers: 4, interesting: 15
fuzzing, elapsed: 12.0s, execs: 1400 (117/sec), workers: 4, interesting: 15
...
```

需要注意的是：
- Fuzzing 会消耗大量的内存，在运行时会影响到机器的性能（一运行，小风扇就转了起来）。
- Fuzzing 会默认使用 `GOMAXPROCS`相同的核数，可以通过执行 `-parallel` 标识来控制数量。
- Fuzzing 会默认在运行时，将扩大测试范围的数值写入 `$GOCACHE/fuzz` 内的模糊缓存目录，目前是没有限制的，可以通过运行 `gotip clean -fuzzcache` 来清除。

## 总结

在今天这篇文章中，我们介绍了 Fuzzing 是什么。
简单而言，模糊测试（Fuzzing）在真实环境已经被验证了其有效性，其可以随机生成测试用例去覆盖人为测不到的各种复杂场景，带来很大的收益。

在接下来中，除了依赖开源的 go-fuzz 库外，Go 语言也正式的在支持 Fuzzing，虽然他放了 Go1.17 的鸽子...

这会对构建 Go 程序健壮性的又一强心剂！