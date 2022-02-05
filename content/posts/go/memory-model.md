---
title: "Go 内存模型：happens-before 原则"
date: 2021-12-31T12:54:54+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

在日常工作中，如果我们能够了解 Go 语言内存模型，那会带来非常大的作用。这样在看一些极端情况，又或是变态面试题的时候，就能够明白程序运行表现下的很多根本原因了。

当然，靠一篇普通文章讲完 Go 内存模型，不可能。因此今天这篇文章，把重点划在给大家**讲解 Go 语言的 happens-before 原则**这 1 个细节。

开吸，和煎鱼揭开他的神秘面纱！

内存模型定义是什么
---------

既然要了解 happens-before 原则，我们得先知道 The Go Memory Model（Go 内存模型）定义的是什么，官方解释如下：

> The Go memory model specifies the conditions under which reads of a variable in one goroutine can be guaranteed to observe values produced by writes to the same variable in a different goroutine.

在 Go 内存模型规定：“在一个 goroutine 中读取一个变量时，可以保证观察到不同 goroutine 中对同一变量的写入所产生的值” 的条件。

这是学习后续知识的一个大前提。

happens-before 是什么
------------------

Happens Before 是一个专业术语，与 Go 语言没有直接关系，也就是并非是特有的。用大白话来讲，其定义是：

>  在一个多线程程序中，假设存在 A 和 B 两个操作，如果 A 操作在 B 操作之前发生（A happens-before B），那么 A 操作对内存的影响将会对执行 B 的线程可见。

A 不一定 happens-before B
----------------------

从 happens-before 定义来看，我们可以反过来想。那就是：

>  在同一个（相同）线程中，如果都执行 A 和 B 操作，并且 A 的声明一定在 B 之前，那么 A 一定先于（happens-before）B 发生。

以下述 Go 代码例子：

```
var A int
var B int

func main() {
 A = B + 1  (1)
 B = 1      (2)
}
```

该代码是在同一个 main goroutine，全局变量 A 在变量 B 之前声明。

在 main 函数中，代码行 (1)，也在代码行 (2) 之前。因此我们可以得出 (1) 一定会在 (2) 前执行，对吗？

答案是：错误的，因为 A happens-before B 并不意味着 A 操作一定会在 B 操作之前发生。

实际上在编译器中，上述代码在汇编的真正执行顺序如下：

```
 0x0000 00000 (main.go:7) MOVQ "".B(SB), AX
 0x0007 00007 (main.go:7) INCQ AX
 0x000a 00010 (main.go:7) MOVQ AX, "".A(SB)
 0x0011 00017 (main.go:8) MOVQ $1, "".B(SB)
```

*   (2)：加载 B 到寄存器 AX。
    
*   (2)：进行 B = 1 赋值，在代码中执行为 INCQ 自增。
    
*   (1)：将寄存器 AX 中值加上 1 后赋值给 A。
    

通过上述分析，我们可以得知。在代码行 (1) 在 (2) 之前，但确实 (2) 比 (1) 更早执行。

那么这是不是意味着违反了 happens-before 的设计原则，毕竟这可是同个线程里的操作，Go 编译器有 BUG？

其实不然，因为对 A 的赋值实质上对 B 的赋值没有影响。所以并没有违反 happens-before 的设计原则。

Go 语言中的 happens-before
----------------------

在 《The Go Memory Model》 中，给出了 Go 语言中 Happens Before 的明确语言定义。

以下术语将会在介绍中用到：

*   变量 v：一个指代性的变量，用于示例演示。
    
*   读 r：代表读操作。
    
*   写 w：代表写操作。
    

### 定义

在满足如下两点条件下，允许对变量 v 的读 r 观察对 v 的写 w：

1.  r 在 w 之前没有发生。
    
2.  没有其他写到 v 的 w' 发生在 w 之后但在 r 之前。
    

为了保证变量 v 的读 r 观察到对 v 的特定写 w，确保 w 是唯一允许 r 观察的写。

因此如果以下两点都成立，就能保证 r 能观察到 w ：

1.  w 发生在 r 之前。
    
2.  对共享变量 v 的任何其他写入都发生在 w 之前或 r 之后。
    

这看起来比较生涩，接下来我们以《The Go Memory Model》 中具体的 channel 例子来进行进一步说明，会更好理解一些。

Go Channel 实例
-------------

在 Go 语言中提倡不要通过共享内存来进行通讯；相反，应当通过通讯来共享内存：

> Do not communicate by sharing memory; instead, share memory by communicating.

因此在 Go 工程中，Channel 是一个非常常用的语法。在原则上其需要遵守：

1.  一个 channel 上的发送是在该 channel 的相应接收完成之前发生的。
    
2.  channel 的关闭发生在接收之前，因为通道被关闭而返回一个零值。
    
3.  一个无缓冲 channel 的接收发生在该 channel 的发送完成之前。
    
4.  一个容量为 C 的 channel 上，第 k 次接收发生在该 channel 的第 k+C 次发送完成之前。
    

接下来根据这四条原则，我们逐一给出例子，用于学习和理解。

例子 1
----

Go channel 例子 1，你认为输出的结果是什么。如下：

```
var c = make(chan int, 10)
var a string

func f() {
 a = "炸煎鱼"   (1)
 c <- 0        (2)
}

func main() {
 go f()
 <-c           (3)
 print(a)      (4)
}

```

答案是空字符串吗？

程序最终结果是正常输出 “炸煎鱼” 的，原因如下：

*   (1) happens-before (2) 。
    
*   (4) happens-after (3)。
    

当然，最后 (1) 写入变量 a 的操作，必然 happens-before 于 (4) print 方法，因此正确的输出了 “炸煎鱼”。

能够满足 “一个 channel 上的发送是在该 channel 的相应接收完成之前发生的”。

例子 2
----

主要是确保了关闭管道时的行为。只需要在前面的例子中，替换 `c <- 0` 成 `close(c)` 就能够产生具有相同的行为保证的程序。

能够满足 “channel 的关闭发生在接收之前，因为通道被关闭而返回一个零值”。

例子 3
----

Go channel 例子 3，你认为输出的结果是什么。如下：

```
var c = make(chan int)
var a string

func f() {
 a = "煎鱼进脑子了"    (1)
 <-c                 (2)
}

func main() {
 go f()
 c <- 0              (3)
 print(a)            (4)
}

```

答案是空字符串吗？

程序最终结果是正常输出 “煎鱼进脑子了” 的，原因如下：

*   (2) happens-before (3)。
    
*   (1) happens-before (4)。
    

能够满足 “一个无缓冲 channel 的接收发生在该 channel 的发送完成之前”。

如果我们把无缓冲改为 `make(chan int, 1)`，也就是带缓冲的 channel，则无法保证正常的输出 “煎鱼进脑子了”。

例子 4
----

Go channel 例子 4，这个程序为工作列表中的每个条目启动一个 goroutine，但 goroutine 使用 channel 进行协调，以确保每次最多只有三个工作函数在运行。

代码如下：

```
var limit = make(chan int, 3)

func main() {
 for _, w := range work {
  go func(w func()) {
   limit <- 1
   w()
   <-limit
  }(w)
 }
 select{}
}
```

能够满足 “一个容量为 C 的 channel 上，第 k 次接收发生在该 channel 的第 k+C 次发送完成之前”。

总结
--

在本文中，我们针对 happens-before 原则进行了基本的说明。同时结合 Go 语言中实际的 happens-before 和 happens-after 的场景进了展示和讲解。

实际上，在日常的开发工作中，happens-before 原则基本已经深入到潜意识中，就跟设计模式一样。会不知觉就应用到，但是若我们希望更进一步的对 Go 语言等内存模型就行研究和理解，就必须对这个基本理念有所认知。

你平时有没有注意到这块的问题呢，欢迎大家留言和讨论！

## 鼓励

若有任何疑问欢迎评论区反馈和交流，**最好的关系是互相成就**，各位的**点赞**就是[煎鱼](https://github.com/eddycjy)创作的最大动力，感谢支持。

> 文章持续更新，可以微信搜【脑子进煎鱼了】阅读，本文 **GitHub** [github.com/eddycjy/blog](https://github.com/eddycjy/blog) 已收录，学习 Go 语言可以看 [Go 学习地图和路线](https://github.com/eddycjy/go-developer-roadmap)，欢迎 Star 催更。

参考
--

*   The Go Memory Model
    
*   Go内存模型&Happen-Before（一）
    
*   GoLang 内存模型
    
*   Golang happens before & channel
    
*   Go 内存模型
