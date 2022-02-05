---
title: "Go1.17 新特性：对 Go 依赖管理的一把大剪刀"
date: 2021-12-31T12:55:03+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

不得不说。我可是个经历过 Go 依赖管理群魔乱舞，Go modules 迁移一堆 BUG 的人儿，难顶...
为此当年我写了不少技术文章，希望给大家避坑。

如下：
- [Go Modules 终极入门](https://mp.weixin.qq.com/s/6gJkSyGAFR0v6kow2uVklA)
- [干货满满的 Go Modules 知识分享](https://mp.weixin.qq.com/s/uUNTH06_s6yzy5urtjPMsg)
- [Go1.16 新特性：Go mod 的后悔药，仅需这一招](https://mp.weixin.qq.com/s/0g89yj9sc1oIz9kS9ZIAEA)

在近期 Go1.17 发布后，Go modules 带来了两大更新，煎鱼摩拳擦掌，他们分别是：
- 模块依赖图裁剪（module graph pruning）
- 延时模块加载（lazy module loading）

今天带大家一起来了解这两块内容，争取了解其为何物，背景又是什么。

## 背景

在日常的 Go 工程开发中，不知道你有没有遇到过 Go modules 的一个奇怪的点。大家没说，就以为是正确的，默认就接受了。

引用官方的 [mod_lazy_new_import.txt](https://github.com/golang/go/blob/4012fea822763ef3aa66dd949fa95b9f8d89450a/src/cmd/go/testdata/script/mod_lazy_new_import.txt "mod_lazy_new_import.txt") 的案例来说，就是假设我们在代码中：
- main module 是 lazy。其导入了 module A 的 package x，package x 又导入了 module B。
- main module lazy 也相当于同时导入了 module A 的 package y。
- module A 的 package y 又导入了 module C 的 package z。

关联如下图所示：

![module 关联图示](https://image.eddycjy.com/d535e2d41648a346c08c41ac38eff6b9.jpg)

这个 Go 程序如果运行起来，会发生什么情况呢？**在 Go1.17 以前，如果你不存在 module C 的 package z**，程序在编译构建的时候就会报错，提示找不到。

实际上 module C 的 package z 并没有**对你主程序有任何建设意义**，俗话来讲就是 “占着茅坑不拉屎”。

他只是因为 main module 在导入 module A 时，也被 “间接” 导入了 package y 的依赖，也就是我们常看到的 go.mod 文件中的 “indirect” 标识，他们会导致构建失败，让人直呼无奈。

## Go1.17 module 改进

显然，社区反馈希望避免看到 “不相关” 的传递依赖等，也因此有了 Go1.17 的 module 改造。

![](https://files.mdnice.com/user/3610/e69f5155-3334-462f-b021-bee477c62c49.png)

接下来的 module 例子我们将会结合提案 [《Proposal: Lazy Module Loading》](https://go.googlesource.com/proposal/+/master/design/36460-lazy-module-loading.md "Proposal: Lazy Module Loading") 、[《cmd/go: module graph pruning and lazy module loading》](https://github.com/golang/go/issues/36460 "cmd/go: module graph pruning and lazy module loading") 以及 《[Module graph pruning](https://golang.google.cn/ref/mod#graph-pruning "Module graph pruning")》的内容、案例来进行说明和介绍。

### module graph pruning

第一个改进就是模块依赖图裁剪（module graph pruning），这是这个版本 module 优化的基础。

在 Go1.17 以前，只要该项目的 go.mod 文件分析出来你存在间接的依赖，如果你没有安装过该依赖，就会出现报错。

错误提示如下：

```golang
$ go build
go: example.com/a@v0.1.0 requires
 example.com/c@v0.1.0: missing go.sum entry; to add it:
 go mod download example.com/c
```

这个时候我们都会默默地去安装一遍...没想过这是间接依赖，和我们的程序没一点直接的代码关系。

在 Go1.17 及之后就变了，go.mod 文件如下，会存在 2 块 require 代码块：

```golang
module example.com/lazy

go 1.17

require example.com/a v0.1.0

require example.com/b v0.1.0 // indirect
...
```

这就是区别，第一块的 require 我们眼熟，那分拆出来的第二块 require 的是什么呢？

这就是那些模块的间接依赖（常见到的 indirect 标识依赖）。可以理解为像是其他语言的 xxx.lock 文件一样的存在。

此处分析出来的间接依赖，将会不会像以前一样阻碍编译构建，只会真正有使用到的才会进行识别。

### lazy module loading

第二个改进是延时模块加载（lazy module loading），是基于模块依赖图裁剪（module graph pruning）的场景上的进一步优化。

也就是以往那些没被使用到的，但又间接依赖的模块。在 Go1.17 及以后不会被 Go 命令读取和加载，只有真正需要的时候才会加载。

### 副作用

Go module 依赖图裁剪也带来了一个副作用，那就是 go.mod 文件 size 会变大。

在 Go 1.17 版本之后，每次 go mod tidy（当go.mod中的go版本为1.17时），Go 命令都会对 main module 的依赖做一次深度扫描（deepening scan）。

该操作将 main module 的所有直接和间接依赖都记录在 go.mod 中，考虑到内容较多，Go 1.17 将直接依赖和间接依赖分别放在两个不同的 require 代码块中。

也就是上文所见到的内容。

## 总结

自从 Go 语言推出 Go modules 依赖，module 一直不断地在优化和改进。虽然看上去已经越来越好用了，但依然似乎存在不少问题。

就拿本次变更来讲，我也是在好朋友的 Go 微信群中看到提问，才思考了起来。因为大家看到第二块 require 时，虽然知道是间接依赖的包，但更明确，为什么要单独出来？

大家其实是不大理解的，本次变更也可能存在语义不清，不够明确的情况。但无论如何，后续我们可以继续观察。