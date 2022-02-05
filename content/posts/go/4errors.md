---
title: "大家对 Go 错误处理的 4 个误解！"
date: 2021-12-31T12:55:04+08:00
draft: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

Go 语言中错误处理的机制一直是各大 Gopher 热议的问题。甚至一直有人寄望 Go 支持 `throw` 和 `catch` 关键字，实现与其他语言类似的特性。

今天煎鱼带大家了解几个 Go 语言的错误处理中，大家最关心的问题：
1. 为什么不支持 try-catch？
2. 为什么不支持全局捕获的机制？
3. 为什么要这么设计错误处理？
4. 未来的错误处理机制会怎么样？

## 落寞的 try-catch

在 Go1 时，大家知道基本不可能支持。于是打起了 Go2 的主意。为什么 Go 就不能支持 try-catch 组合拳？

上一年宣发了 Go2 的构想，所以在 2020 年就有小伙伴乘机提出过类似 《[proposal: Go 2: use keywords throw, catch, and guard to handle errors](https://github.com/golang/go/issues/40583 "proposal: Go 2: use keywords throw, catch, and guard to handle errors")》的提案。

下面来自该提案的演示，Go1 的错误处理： 

```golang
type data struct {}

func (d data) bar() (string, error) {
    return "", errors.New("err")
}

func foo() (data, error) {
    return data{}, errors.New("err")
}

func do () (string, error) {
    d, err := foo()
    if err != nil {
        return "", err
    }

    s, err := d.bar()
    if err != nil {
        return "", err
    }

    return s, nil
}
```

新提案所改造的方式：

```golang
type data struct {}

func (d data) bar() string {
    throw "", errors.New("err")
}

func foo() (d data) {
    throw errors.New("err")
    return
}

func do () (string, error) {
    catch err {
        return "", err 
    }

    s := foo().bar()
    return s, nil
}
```

不过答复非常明确，@davecheney 在底下回复“以最强烈的措辞，不（In the strongest possible terms, no）”。这可让人懵了圈，为什么这么硬呢？

其实 Go 官方早在《[Error Handling — Problem Overview](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md "Error Handling — Problem Overview")》提案早已明确提过，Go 官方在**设计上会有意识地选择使用显式错误结果和显式错误检查**。

结合《[language: Go 2: error handling meta issue](https://github.com/golang/go/issues/40432 "language: Go 2: error handling meta issue")》可得知，要拒绝 try-catch 关键字的主要原因是：
- 会**涉及到额外的流程控制**，因为使用 try 的复杂表达式，会导致函数意外返回。
- 在表达式层面上没有流程控制结构，只有 panic 关键字，它不只是从一个函数返回。

说白了，就是设计理念不合，加之实现上也不大合理。在以往的多轮讨论中早已被 Go 团队拒绝了。

反之 Go 团队倒是一遍遍在回答这个问题，已经不大耐烦了，直接都整理了 issues 版的 FAQ 了。

## 想捕获所有 panic

在 Go 语言中，有一个点，很多新同学会不一样碰到的。那就是在 goroutine 中如果 panic 了，没有加 recover 关键字（有时候也会忘记），就会导致程序崩溃。

又或是以为加了 recover 就能保障一个 goroutine 下所派生出来的 goroutine 所产生的 panic，一劳永逸。

但现实总是会让人迷惑，我经常会看到有同学提出类似的疑惑：

![来自 Go 读者交流群](https://files.mdnice.com/user/3610/8e3127ef-7cbe-4952-8780-f8812f921f45.png)

这时候，有其他语言经验的同学中，又有想到了一个利器。能不能设置一个全局的错误处理 handler。

像是 PHP 语言也可以有类似的方法：

```
set_error_handler();
set_exception_handler();
register_shutdown_function();
```

显然，Go 语言中并没有类似的东西。归类一下，我们聚焦以下两个问题：
1. 为什么 recover 不能捕获更上层的 panic？
2. 为什么 Go 没有全局的错误处理方法？

### 源码层面

如果是讲设计的话，其实只是通过 Go 的 GMP 模型和 defer+panic+recver 的源码剖析就能知道了。

![](https://files.mdnice.com/user/3610/42998ba0-84cc-4fe5-b88d-b9400dd1698b.png)

本质上 defer+panic 都是挂载在 G 上的，可查看我以前写的《[深入理解 Go panic and recover](https://eddycjy.com/posts/go/panic/2019-05-21-panic-and-recover/ "深入理解 Go panic and recover")》，你会有更多深入的理解。

### 设计思想

在本文中我们不能仅限于源码，需要更深挖，Go 设计者他的思想是什么，为什么就是不支持？

在 Go issues 中《[proposal: spec: allow fatal panic handler](https://github.com/golang/go/issues/32333 "proposal: spec: allow fatal panic handler")》、《[No way to catch errors from goroutines automatically](https://github.com/golang/go/issues/20161 "No way to catch errors from goroutines automatically") 》分别的针对性探讨过上述问题。

Go 团队的大当家 @Russ Cox 给出了明确的答复：**Go 语言的设计立场是错误恢复应该在本地完成，或者完全在一个单独的进程中完成**。

![](https://files.mdnice.com/user/3610/cb6fac34-ba12-40fd-98d3-f177e00ee39f.png)

这就是为什么 Go 语言不能跨 goroutines 从 panic 中恢复，也不能从 throw 中恢复的根本原因，是**语言设计层面的思想所决定**。

在源码剖析时，你所看到的整套 GMP+defer+panic+recover 的机制机制，就是跟随着这个设计思想去编写和发展的。

设计思想决定源码实现。

### 建议方式

从 Go 语言层面去动摇这个设计思想，目前来看可能性很低。至少 2021 年的现在没有看到改观。

整体上会建议提供公共的 Go 方法去规避这种情况。参考 issues 所提供的范例如下：

```golang
recovery.SafeGo(logger, func() {
              method(all parameters)
	})

func SafeGo(logger logging.ILogger, f func()) {
	go func() {
		defer func() {
			if panicMessage := recover(); panicMessage != nil {
				...
			}
		}()

		f()
	}()
}
```

是不是感觉似曾相识？

每家公司的内部库都应该有这么一个工具方法，规避偶尔忘记的 goroutine recover 所引发的奇奇怪怪问题。

也可以参照建议，利用一个单独的进程（Go 语言中是 goroutine）去统一处理这些 panic，不过这比较麻烦，较少见。

## 未来会如何

Go 社区对 Go 语言未来的错误处理机制非常关心，因为 Go1 已经米已成炊，希望在 Go2 上解决错误处理机制的问题。

期望 Go2 核心要处理的包含如下几点（#40432）：

1. 对于 Go2，我们希望使错误检查更加轻便，减少专门用于错误检查的 Go 程序代码的数量。我们还想让写错误处理更方便，减少程序员花时间写错误处理的可能性。
2. 错误检查和错误处理都必须保持明确，即在程序文本中可见。我们不希望重复异常处理的陷阱。
3. 现有的代码必须继续工作，并保持与现在一样的有效性。任何改变都必须与现有的代码相互配合。

为此，许多人提过不少新的提案...很可惜，截止 2021.08 月底为止，有许多人试图改变语言层面以达到这些目标，但没有一个新的提案被接受。

现在也有许多变更并入 Go2 提案，主要是 error-handling 方面的优化。

大家有兴趣可以看看我之前写的：《[先睹为快，Go2 Error 的挣扎之路](https://mp.weixin.qq.com/s/XILveKzh07BOQnqxYDKQsA)》，相信能给你带来不少新知识。

## 总结

看到这里，我们不由得想到。为什么，为什么在 21 世纪前者已经有了这么多优秀的语言，Go 语言的错误处理机制依然这么的难抉择？

显然 Go 语言的开发团队是有自己的设计哲学和思想的，否则 “less is more” 也不会如此广泛流传。

这存在着一系列既要也要的问题。欢迎大家关注煎鱼，后续我们也可以面向 Go 后续的错误处理持续的关注和讨论！