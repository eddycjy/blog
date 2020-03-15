---

title:      "Go1.12 defer 会有性能损耗，尽量不要用？"
date:       2019-06-16 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
---

![image](https://s2.ax1x.com/2020/02/27/3wuUYV.jpg)

上个月在 @polaris @轩脉刃 的全栈技术群里看到一个小伙伴问 **“说 defer 在栈退出时执行，会有性能损耗，尽量不要用，这个怎么解？”**。

恰好前段时间写了一篇 [《深入理解 Go defer》](https://segmentfault.com/a/1190000019303572) 去详细剖析 `defer` 关键字。那么这一次简单结合前文对这个问题进行探讨一波，希望对你有所帮助，但在此之前希望你花几分钟，自己思考一下答案，再继续往下看。

## 测试

```go
func DoDefer(key, value string) {
    defer func(key, value string) {
        _ = key + value
    }(key, value)
}

func DoNotDefer(key, value string) {
    _ = key + value
}
```

基准测试：

```go
func BenchmarkDoDefer(b *testing.B) {
    for i := 0; i < b.N; i++ {
        DoDefer("煎鱼", "https://github.com/EDDYCJY/blog")
    }
}

func BenchmarkDoNotDefer(b *testing.B) {
    for i := 0; i < b.N; i++ {
        DoNotDefer("煎鱼", "https://github.com/EDDYCJY/blog")
    }
}
```

输出结果：

```
$ go test -bench=. -benchmem -run=none
goos: darwin
goarch: amd64
pkg: github.com/EDDYCJY/awesomeDefer
BenchmarkDoDefer-4          20000000            91.4 ns/op        48 B/op          1 allocs/op
BenchmarkDoNotDefer-4       30000000            41.6 ns/op        48 B/op          1 allocs/op
PASS
ok      github.com/EDDYCJY/awesomeDefer 3.234s
```

从结果上来，使用 `defer` 后的函数开销确实比没使用高了不少，这损耗用到哪里去了呢？

## 想一下

```
$ go tool compile -S main.go
"".main STEXT size=163 args=0x0 locals=0x40
    ...
    0x0059 00089 (main.go:6)    MOVQ    AX, 16(SP)
    0x005e 00094 (main.go:6)    MOVQ    $1, 24(SP)
    0x0067 00103 (main.go:6)    MOVQ    $1, 32(SP)
    0x0070 00112 (main.go:6)    CALL    runtime.deferproc(SB)
    0x0075 00117 (main.go:6)    TESTL    AX, AX
    0x0077 00119 (main.go:6)    JNE    137
    0x0079 00121 (main.go:7)    XCHGL    AX, AX
    0x007a 00122 (main.go:7)    CALL    runtime.deferreturn(SB)
    0x007f 00127 (main.go:7)    MOVQ    56(SP), BP
    0x0084 00132 (main.go:7)    ADDQ    $64, SP
    0x0088 00136 (main.go:7)    RET
    0x0089 00137 (main.go:6)    XCHGL    AX, AX
    0x008a 00138 (main.go:6)    CALL    runtime.deferreturn(SB)
    0x008f 00143 (main.go:6)    MOVQ    56(SP), BP
    0x0094 00148 (main.go:6)    ADDQ    $64, SP
    0x0098 00152 (main.go:6)    RET
    ...
```

我们在前文提到 `defer` 关键字其实涉及了一系列的连锁调用，内部 `runtime` 函数的调用就至少多了三步，分别是 `runtime.deferproc` 一次和 `runtime.deferreturn` 两次。

而这还只是在运行时的显式动作，另外编译器做的事也不少，例如：

- 在 `deferproc` 阶段（注册延迟调用），还得获取/传入目标函数地址、函数参数等等。
- 在 `deferreturn` 阶段，需要在函数调用结尾处插入该方法的调用，同时若有被 `defer` 的函数，还需要使用 `runtime·jmpdefer` 进行跳转以便于后续调用。

这一些动作途中还要涉及最小单元 `_defer` 的获取/生成， `defer` 和 `recover` 链表的逻辑处理和消耗等动作。

## Q&A

最后讨论的时候有提到 **“问题指的是本来就是用来执行 close() 一些操作的，然后说尽量不能用，例子就把 defer db.close() 前面的 defer 删去了”** 这个疑问。

这是一个比较类似 “教科书” 式的说法，在一些入门教程中会潜移默化的告诉你在资源控制后加个 `defer` 延迟关闭一下。例如：

```go
resp, err := http.Get(...)
if err != nil {
    return err
}
defer resp.Body.Close()
```

但是一定得这么写吗？其实并不，很多人给出的理由都是 “怕你忘记” 这种说辞，这没有毛病。但需要认清场景，假设我的应用场景如下：

```go
resp, err := http.Get(...)
if err != nil {
    return err
}
defer resp.Body.Close()
// do something
time.Sleep(time.Second * 60)
```

嗯，一个请求当然没问题，流量、并发一下子大了呢，那可能就是个灾难了。你想想为什么？从常见的 `defer` + `close` 的使用组合来讲，用之前建议先看清楚应用场景，在保证无异常的情况下确保尽早关闭才是首选。如果只是小范围调用很快就返回的话，偷个懒直接一套组合拳出去也未尝不可。

## 结论

一个 `defer` 关键字实际上包含了不少的动作和处理，和你单纯调用一个函数一条指令是没法比的。而与对照物相比，它确确实实是有性能损耗，目前延迟调用的全部开销大约在 50ns，但 `defer` 所提供的作用远远大于此，你从全局来看，它的损耗非常小，并且官方还不断地在优化中。

因此，对于 “Go defer 会有性能损耗，尽量不能用？” 这个问题，我认为**该用就用，应该及时关闭就不要延迟，在 hot paths 用时一定要想清楚场景**。

## 补充

最后补充上柴大的回复：**“不是性能问题，defer 最大的功能是 Panic 后依然有效。如果没有 defer，Panic 后就会导致 unlock 丢失，从而导致死锁了”**，非常经典。
