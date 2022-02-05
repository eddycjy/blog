---
title: "Go 插件系统，一个凉了快半截的特性？"
date: 2021-12-31T12:54:59+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

在 Go 语言中，有一个好像很好用，但却比较少人提及的功能，那就是 Go Plugin。目前在 Go 工程中普遍还没广泛的使用起来，覆盖率不高。

前段时间小咸鱼的同事问了他这玩具怎么用，他正想甩出一个链接，但发现...煎鱼竟然没写过，这不，Go 知识板块的文章地图得补全。

今天煎鱼就大家一起学习 Go Plugin！

## 是什么

Go Team 最早在 Go1.7 实验，在 Go1.8 正式引入了 Go Plugin 的机制。于 2016 年发布，一开始仅支持 Linux 实现：

![](https://files.mdnice.com/user/3610/9a791ced-39db-4f09-a027-60569ccc95dd.png)

Go Plugin 机制实现了 Go 插件的加载和符号解析，能够支持将我们所编写的 Go 包编译为共享库（.so）。

这样 Go 工程就可以加载所编译好的 Go Plugin（已经变成了共享库文件），在程序中调用共享库中的函数、常量、变量等使用。也称其为 Go 语言中的热插拔的插件系统。

截止 Go1.17 为止，Go Plugin 仅支持在 Linux、FreeBSD 和 MacOS 上运行，还不支持 Windows。

## 为什么需要

Go 语言是静态语言，正常我们写一个程序，分如下两个角度来看：
- 从代码编写的角度来看：我们在程序编写的时候就已经把所有的功能实现给确定了，不会发生什么根本性的变化。
- 从程序的角度来看：在 Go 进行编译时，就已经把所有引用的标准库、第三方库等都编译打包好进二进制文件了，因此也就无法在运行时去动态加载，所以没法有其它的可能性。

那么为什么需要 Go Plugin 呢，原因如下：
- 可插拔的插件：程序能够随时的安装插件，也能够卸载他，获得更多运行时的自定义能力。
- 可动态加载运行时模块：随时安装了插件，自然也就需要可自行决定运行哪个插件的模块了。
- 可独立开发插件、模块：主系统和子插件，可能由不同的团队开发和提供，也更有价值。

其实本质上还是**希望程序能够在运行时实现动态的外部加载**，根据不同的条件、场景加载不同的插件功能。

## 使用方法

### 通用概念

Go 官方给出的例子非常简单，只需要在 Go 编译时指定为插件就可以了。

编译的命令例子如下：

```golang
go build -buildmode=plugin
```

当一个插件初次被打开时，所有尚未成为程序一部分的包的init函数被调用。不过主函数不被运行。需要注意**一个插件只会被初始化一次，插件不能被关闭**。

其共有如下几个 API：

```golang
type Plugin
    func Open(path string) (*Plugin, error)
    func (p *Plugin) Lookup(symName string) (Symbol, error)
type Symbol
```
- Plugin.Open：开启一个 Go 插件。如果一个路径已经被打开，那么将返回现有的 `*Plugin`。
- Plugin.Lookup：在插件中搜索名所传入的符号，符号是任何导出的变量或函数。如果没有找到该符号，它会报告一个错误。

主要就是细分为插件和符号，符号（Symbol）本身是一个 interface，在调用 Plugin 相关方法后还是需要进一步断言才能使用。

### 实际编写

了解基本定义后，我们定义一个插件，一般我们会有个 plugins/ 的目录，作为主程序的附属插件集。

插件的代码如下：

```golang
package main

import "fmt"

var V int

func F() {
	fmt.Printf("脑子进了 %d 次煎鱼 \n", V)
}
```

包名必须为 main，在该插件根目录运行： 

```golang
go build -buildmode=plugin -o plugin.so main.go
```

就可以看到在编译的目录下多出了 `plugin.so` 文件，这就是这个插件经过编译后的动态库 .so 文件。

随后只需在主程序加载这个插件就可以了，如下：

```golang
import (
	"plugin"
)

func main() {
	p, err := plugin.Open("plugin.so")
	if err != nil {
		panic(err)
	}
	v, err := p.Lookup("V")
	if err != nil {
		panic(err)
	}
	f, err := p.Lookup("F")
	if err != nil {
		panic(err)
	}

	*v.(*int) = 999
	f.(func())()
}
```

输出结果：

```
脑子进了 999 次煎鱼 
```

在程序中，我们先调用了 `plugin.Open` 方法打开了前面所编译的 `plugin.so` 动态库。

紧接着调用 `plugin.Lookup` 方法，定位到了变量 V 和 方法 F，但由于其返回值都是 Symbol（interface），因此我们需要对其进行类型断言，随时才可以调用和使用。

至此完成了一个插件的基本使用。

## 为什么不被需要

在前面我们提到了大量 Go Plugin 的优点，也演示了其 Plugin 代码编写起来有多么的简单和方便。

但，**为什么 Go Plugin 已经发布了 4 年依然没有被大规模应用**，甚至对于不少业务开发来讲是不被需要的呢，或是压根不知道有这东西？

究其原因，我个人认为一个东西的广泛应用要至少符合以下三大点：
- 基数：需要的场景多。
- 上手：方便且易用。
- 质量：没有大问题。

比较折腾的人的是，Go Plugin 这三大点都欠一些火候，综合导致了该功能的没有大规模应用。

像是要应用 Go Plugin 有诸如下约束：

- 环境问题：不支持 Windows 等（暂无计划，#19282），MacOS 有些问题，一开始只支持 Linux，其他的也是后面慢慢增加的支持。
- Go 版本问题：Plugin 构建环境和第三方包的依赖版本需要保持一致。
- 特性问题：Plugin 特性的缺失，例如不支持插件的关闭，暂时无新计划支持（#20461）。

## 总结

在 Go issues 中畅游时，能看到许多小伙伴在以往 4 年踩过的坑和无奈。甚至有一个高赞回答（#19282）表示：**插件功能主要是一个技术演示，由于一些不道德的原因，被作为语言的稳定功能发布**（The plugin feature is mostly a tech demo that for some unholy reason got released as a stable feature of the language.）。

目前 Go Plugin 并不是 Go Team 的优先事项，在 Windows/Mac 的支持存在问题。GOPATH 有问题，不同 GO 版本也有问题。更是建议如果您想要插件，请走较慢的 grpc 路线，因为它们是有效的插件。

也可以参考为数不多的一些 Go Plugin 用户的方案，例如：tidb。

但如果要正式使用，是需要慎重考虑，又或是再等等...等更完善的那一天？

## 参考

- [Go Package plugin](https://golang.org/pkg/plugin/)
- [Why is there no windows support for plugins?](https://www.reddit.com/r/golang/comments/bsoa4e/why_is_there_no_windows_support_for_plugins/)
- [plugin: add Windows support](https://github.com/golang/go/issues/19282)
- [plugin: Add support for closing plugins](https://github.com/golang/go/issues/20461)
- [如何评价 Go 标准库中新增的 plugin 包？](https://www.zhihu.com/question/51650593)
- [一文搞懂Go语言的plugin](https://mp.weixin.qq.com/s/yBosg0q0V_-wYk9G2GiVTw)