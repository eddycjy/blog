---
title: "Go1.18 新特性：编译后的二进制文件，将包含更多信息"
date: 2022-02-05T16:02:45+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

我有一个朋友，，开开心心入职，想着施展拳脚，第一个任务就是对老旧的二进制文件进行研究。

他一看，这文件，不知道是编译器用什么参数怎么打出来的，环境不知道是什么，更不知道来自什么代码分支？

这除了是项目流程上的问题外，Go 在这块也有类似的小问题，处理起来比较麻烦。

## 背景

日常中很难从 Go 二进制文件中检索元信息，要么是信息完全缺失，要么提取需要对二进制文件进行大量解析。

包含的元信息如下：

|  元信息   | 提取处  |
|  ----  | ----  |
| Go 构建版本  | 符号表，通过全局变量 `runtime.buildVersion` 来获取 |
| 构建信息，例如：模块和版本 | 符号表，通过全局变量 `runtime/debug.modinfo` 来获取 |
| 编译器选项，例如：构建模式、编译器、gcflags、ldflags 等 | 无法获取 |
| 用户定义的自定义数据，例如：应用程序版本等 | 需在编译时设置全局字符串变量，才可以获取 |

关注到编译器选项，也就是参数等都是无法得知的，也就是会提高获取如何编译出来的难度。

## 新提案

Michael Obermüller 提出了一个新的提案《[cmd/go: add compiler flags, relevant env vars to 'go version -m' output](https://github.com/golang/go/issues/35667)》用于解决上述问题。

在提案中想要的是 JSON 格式的结构输出：

```json
{
    "version": "go1.13.4",
    "compileropts": {
        "compiler": "gc",
        "mode": "pie",
        "os": "linux",
        ...
    },
    "buildinfo": {
        "path": "脑子进煎鱼了",
        "main": {
            "path": "HelloWorld",
            "version": "(devel)",
        },
        "deps": []
    },
    "user": {
        "customkey": "customval",
        ...
    }
}
```

Russ Cox 表示由于编译信息已有既有格式，并且默认使用 JSON 只会让二进制文件变得更大。好处少，没必要，改为了选项化的支持。

新的 Go1.18 版本中，可以通过既有的：

```
go version -m
``` 

查看到提案所提到的信息。

例如：

```go
$ gotip version
go version devel go1.18-eba0e866fa Mon Oct 18 22:56:07 2021 +0000 darwin/amd64
$ gotip build ./
$ gotip version -m ko
...
	build	compiler	gc
	build	tags	goexperiment.regabiwrappers,goexperiment.regabireflect,goexperiment.regabiargs
	build	CGO_ENABLED	true
	build	CGO_CPPFLAGS	
	build	CGO_CFLAGS	
	build	CGO_CXXFLAGS	
	build	CGO_LDFLAGS	
	build	gitrevision	6447264ff8b5d48aff64000f81bb0847aefc7bac
	build	gituncommitted	true
```

若需要输出 JSON 格式，也可以通过指定 `go version -json` 达到一样的效果。

在上面的输出中，现有的编译器选项等都会包含在内，能够让大家对整体编译后的二进制文件溯源有一个更好的认知。

## 总结

在今天这篇文章中，给大家介绍了 Go1.18 的一个新的变化。

新版本中，编译器选项/参数、相关环境变量等，将会包含在编译后的二进制文件中，能够更便于后人排查和查看信息。