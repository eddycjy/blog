---
title: "意见征集：Go1 要不要移除 GOPATH？"
date: 2021-04-05T16:02:34+08:00
images:
tags: 
  - go
---

大家好，我是在打自己脸的后方记者煎鱼。

前几天我发表了文章《意见征集：Go1 要不要移除 GOPATH？》。
本次投票共有 592 人参与了投票。
结果如下：

![](https://static01.imgkr.com/temp/d10e1354bc9d43c68439fb71d8935270.png)

绝大部分人支持移除 GOPATH，但建议保留的人依然占相当一部分。

## 最新进展

在近几天，在 go issues 上有人提出了新的提案：《proposal: cmd/go: maintain 'classic' vendor behaviour》。

该提案正式向 Go 官方提出了我们上一篇文章有提到的两点：
1. Go 历史项目的维护问题。
2. Go1 兼容性保证的许诺。

摘选一部分核心观点给大家看看。

## 新的提案

从 Go1.17 开始，Go1.17 就将 GOPATH 从编译器工具链中移除 `GO111MODULE` 标识。

其依据如下：

![](https://static01.imgkr.com/temp/bf314e0979b7481681c6293ee60715c3.png)

重点在于执行 `-mod=vendor` 命令时，其会忽略主模块根目录以外位置的软件目录，**这意味着在 go mod 之前设计和编写的应用程序不能再编译（指的历史项目问题）**。

提案本身并不是要求要保持 GOPATH 代码解析 1:1 的说法，而是希望能够允许项目代码通过其他的方式在移除 GOPATH 后也能正常运行，有其他方式能够导入（不需要转成 Go mod）。

另外最后提到删除 GOPATH 这样的代码解析能力，**与下面这段摘自 go1compat 文档的精神不一致**：

>> It is intended that programs written to the Go 1 specification will continue to compile and run correctly, unchanged, over the lifetime of that specification. At some indefinite point, a Go 2 specification may arise, but until that time, Go programs that work today should continue to work even as future "point" releases of Go 1 arise (Go 1.1, Go 1.2, etc.).

指出直接移除 GOPATH 可能违反 Go1 兼容性保证的精神，带来历史项目的工作量。

## 相爱相杀

无独有偶，前几天我和欧神（@changkun）就讨论过 Go1 兼容性保证（go1compat）的问题，我也是认为违反了该精神保证，这次行为不是很合理。

但...欧神重读 《Go 1 and the Future of Go Programs》后，也就是我们俗称的 Go1 兼容性保证。发现了以下亮点：

![](https://static01.imgkr.com/temp/aa47a80786814576b5a5b091d8244127.png)

在工具链中提到，**Go 语言的工具链（编译器、链接器、构建工具等）仍然正在积极开发中，可能会改变行为，也就是不在兼容性保护的范围内**。

我们回到主角身上，GOPATH 是什么：GOPATH 定义的是软件的构建形式而不是编译的条件，归属于工具链里。

因此其实**移除 GOPATH 并不违反 Go1 兼容性保证**，因为他不在保障范围内。

注：感谢欧神的探讨和释疑，在此推荐欧神的大作《[Go 语言原本](https://golang.design/under-the-hood/ "Go 语言原本")》。

## 总结

目前该[提案](https://github.com/golang/go/issues/44519 "提案")已经进入 proposal review meeting 日程了，很快就会有决策了，在这里我们就不进行过多的新方案探讨。

现实场景中依赖 GOPATH 的历史项目确实存在，使用什么方法更低成本、合理的让既有项目能够正常运行将会是一个讨论重点。

接下来煎鱼将会跟踪这个提案，新的消息会继续同步。毕竟我是利益相关者（有依赖 GOPATH 的项目)...

欢迎大家一起来讨论各种可能性！