---
title: "Go1.18 新特性：多 Module 工作区模式"
date: 2022-02-05T16:00:00+08:00
toc: true
images:
tags: 
  - go
  - go1.18
---

大家好，我是煎鱼。

Go 的依赖管理，也就是 Go Module。从推出到现在，也已经有了一定的年头了，吐槽一直很多，官方也不断地在进行完善。

Go1.18 将会推出一个新特性：Multi-Module Workspaces，用于支持 Module 多工作区，能解决以往的一系列问题。

今天将由煎鱼带大家一起深入学习。

## 背景

在日常使用 Go 工程时，总会遇到 2 个经典问题，特别的折腾人。

如下：
1. 依赖本地 replace module。
2. 依赖本地未发布的 module。

### replace module

第一个场景：像是平时在 Go 工程中，我们为了解决一些本地依赖，或是定制化代码。会在 go.mod 文件中使用 replace 做替换。

如下代码：

```go
replace golang.org/x/net => /Users/eddycjy/go/awesomeProject
```

这样就可以实现本地开发联调时的准确性。

问题就在这里：
- 本地路径：所设定的 replace 本质上转换的是本地的路径，也就是每个人都不一样。
- 仓库依赖：文件修改是会上传到 Git 仓库的，不小心传上去了，影响到其他开发同学，又或是每次上传都得重新改回去。

用户体验非常差，很折腾人。

### 未发布的 module

第二个场景：在做本地的 Go 项目开发时，可能会在本地同时开发多个库（项目库、工具库、第三方库）等。

如下代码：

```go
package main

import (
    "github.com/eddycjy/pkgutil"
)

func main() {
    pkgutil.PrintFish()
}
```

如果这个时候运行 `go run` 或是 `go mod tidy`，都不行，会运行失败。

报如下类似错误：

```
fatal: repository 'https://github.com/eddycjy/pkgutil/' not found
```

这个问题报错是因为 `github.com/eddycjy/pkgutil` 这个库，在 GitHub 是没有的，自然也就拉取不到。

解决方法：在 Go1.18 以前，我们会通过 replace（会遇到背景一的问题），又或是直接上传到 Github 上，自然也就能被 Go 工具链拉取到依赖了。

许多同学会发出灵魂质疑：Go 的依赖都必须要上传到 GitHub 吗，强绑定？

对新入门的同学非常不友好，很要命。

## 工作区模式

在社区的多轮反馈下，Michael Matloob 提出了提案《[Proposal: Multi-Module Workspaces in cmd/go](https://go.googlesource.com/proposal/+/master/design/45713-workspace.md "Proposal: Multi-Module Workspaces in cmd/go")》进行了大量的讨论和实施，在 Go1.18 正式落地。

新提案的一个核心概念，就是增加了 `go work` 工作区的概念，针对的是 Go Module 的依赖管理模式。

其能够在本地项目的 go.work 文件中，通过设置一系列依赖的模块本地路径，再将**路径下的模块组成一个当前的工作区**，他的读取优先级是最高的。

我们可以通过 `go help` 来查看，如下：

```
$ go1.18beta1 help work
Usage:

	go work <command> [arguments]

The commands are:

	edit        edit go.work from tools or scripts
	init        initialize workspace file
	sync        sync workspace build list to modules
	use         add modules to workspace file

Use "go help work <command>" for more information about a command.
```

只要执行 `go work init` 就可以初始化一个新的工作区，后面跟的参数就是要生成的具体子模块 mod。

命令如下：

```go
go work init ./mod ./tools
```

项目目录如下：

```
awesomeProject
├── mod
│   ├── go.mod      // 子模块
│   └── main.go
├── go.work         // 工作区
└── tools
    ├── fish.go
    └── go.mod      // 子模块
```

生成的 go.work 文件内容：

```
go 1.18

use (
    ./mod 
    ./tools
)
```

新的 go.work 与 go.mod 语法一致，也可以使用 replace 语法：

```
go 1.18

use (...)

replace golang.org/x/net => example.com/fork/net v1.4.5
```

go.work 文件内共支持三个指令：
- go：声明 go 版本号，主要用于后续新语义的版本控制。
- use：声明应用所依赖模块的具体文件路径，路径可以是绝对路径或相对路径，可以在应用命目录外均可。
- replace：声明替换某个模块依赖的导入路径，优先级高级 go.mod 中的 replace 指令。

若想要禁用工作区模式，可以通过 `-workfile=off` 指令来指定。

也就是在运行时执行如下命令：

```
go run -workfile=off main.go

go build -workfile=off
```

go.work 文件是不需要提交到 Git 仓库上的，否则就比较折腾了。

只要你在 Go 项目中设置了 go.work 文件，那么在运行和编译时就会进入到工作区模式，会优先以工作区的配置为最高优先级，来适配本地开发的诉求。

至此，工作区的核心知识就介绍完毕了。

## 总结

今天给大家介绍了 Go1.18 的新特性：多 Module 工作区模式。其本质上还是为了解决本地开发的诉求。

由于 go.mod 文件是与项目强关联的，基本都会上传到 Git 仓库中，很难在这上面动刀子。直接就造了个 go.work 出来，纯本地使用，方便快捷。

使用新的 go.work，就可以在完全是本地文件上各种捣鼓了，不会对其他成员开发有其他影响。

你觉得怎么样呢？：）
