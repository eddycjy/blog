---
title: "跟面试官聊 Goroutine 泄露的 6 种方法，真刺激！"
date: 2021-06-11T12:54:49+08:00
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

前几天分享 Go 群友提问的文章时，有读者在朋友圈下提到，希望我能够针对 Goroutine 泄露这块进行讲解，他在面试的时候经常被问到。

今天的男主角，就是 Go 语言的著名品牌标识 Goroutine，一个随随便便就能开几十万个快车进车道的大杀器。

```golang
    for {
        go func() {}()
    }
```
本文会聚焦于 Goroutine 泄露的 N 种方法，进行详解和说明。

## 为什么要问

面试官为啥会问 Goroutine（协程）泄露这种奇特的问题呢？

可以猜测是：

- Goroutine 实在是使用门槛实在是太低了，随手就一个就能起，出现了不少滥用的情况。例如：并发 map。
- Goroutine 本身在 Go 语言的标准库、复合类型、底层源码中应用广泛。例如：HTTP Server 对每一个请求的处理就是一个协程去运行。

很多 Go 工程在线上出事故时，基本 Goroutine 的关联，大家都会作为救火队长，风风火火的跑去看指标、看日志，通过 PProf 采集 Goroutine 运行情况等。

自然他也就是最受瞩目的那颗 “星” 了，所以在日常面试中，被问几率也就极高了。

## Goroutine 泄露

了解清楚大家爱问的原因后，我们开始对 Goroutine 泄露的 N 种方法进行研究，希望通过前人留下的 “坑”，了解其原理和避开这些问题。

泄露的原因大多集中在：
- Goroutine 内正在进行 channel/mutex 等读写操作，但由于逻辑问题，某些情况下会被一直阻塞。
- Goroutine 内的业务逻辑进入死循环，资源一直无法释放。
- Goroutine 内的业务逻辑进入长时间等待，有不断新增的 Goroutine 进入等待。

接下来我会引用在网上冲浪收集到的一些 Goroutine 泄露例子（会在文末参考注明出处）。

### channel 使用不当

Goroutine+Channel 是最经典的组合，因此不少泄露都出现于此。

最经典的就是上面提到的 channel 进行读写操作时的逻辑问题。

#### 发送不接收

第一个例子：

```golang
func main() {
    for i := 0; i < 4; i++ {
        queryAll()
        fmt.Printf("goroutines: %d\n", runtime.NumGoroutine())
    }
}

func queryAll() int {
    ch := make(chan int)
    for i := 0; i < 3; i++ {
        go func() { ch <- query() }()
	    }
    return <-ch
}

func query() int {
    n := rand.Intn(100)
    time.Sleep(time.Duration(n) * time.Millisecond)
    return n
}
```

输出结果：

```
goroutines: 3
goroutines: 5
goroutines: 7
goroutines: 9
```

在这个例子中，我们调用了多次 `queryAll` 方法，并在 `for` 循环中利用 Goroutine 调用了 `query` 方法。其重点在于调用 `query` 方法后的结果会写入 `ch` 变量中，接收成功后再返回 `ch` 变量。 

最后可看到输出的 goroutines 数量是在不断增加的，每次多 2 个。也就是每调用一次，都会泄露 Goroutine。

原因在于 channel 均已经发送了（每次发送 3 个），但是在接收端并没有接收完全（只返回 1 个 ch），所诱发的 Goroutine 泄露。

#### 接收不发送

第二个例子：

```golang
func main() {
    defer func() {
        fmt.Println("goroutines: ", runtime.NumGoroutine())
    }()

    var ch chan struct{}
    go func() {
        ch <- struct{}{}
    }()
    
    time.Sleep(time.Second)
}
```

输出结果：

```
goroutines:  2
```

在这个例子中，与 “发送不接收” 两者是相对的，channel 接收了值，但是不发送的话，同样会造成阻塞。

但在实际业务场景中，一般更复杂。基本是一大堆业务逻辑里，有一个 channel 的读写操作出现了问题，自然就阻塞了。

#### nil channel

第三个例子：

```golang
func main() {
    defer func() {
        fmt.Println("goroutines: ", runtime.NumGoroutine())
    }()

    var ch chan int
    go func() {
        <-ch
    }()
    
    time.Sleep(time.Second)
}
```

输出结果：

```
goroutines:  2
```

在这个例子中，可以得知 channel 如果忘记初始化，那么无论你是读，还是写操作，都会造成阻塞。

正常的初始化姿势是：

```golang
    ch := make(chan int)
    go func() {
        <-ch
    }()
    ch <- 0
    time.Sleep(time.Second)
```

调用 `make` 函数进行初始化。

### 奇怪的慢等待

第四个例子：

```golang
func main() {
    for {
        go func() {
            _, err := http.Get("https://www.xxx.com/")
            if err != nil {
                fmt.Printf("http.Get err: %v\n", err)
            }
            // do something...
    }()

    time.Sleep(time.Second * 1)
    fmt.Println("goroutines: ", runtime.NumGoroutine())
	}
}
```

输出结果：

```
goroutines:  5
goroutines:  9
goroutines:  13
goroutines:  17
goroutines:  21
goroutines:  25
...
```

在这个例子中，展示了一个 Go 语言中经典的事故场景。也就是一般我们会在应用程序中去调用第三方服务的接口。

但是第三方接口，有时候会很慢，久久不返回响应结果。恰好，Go 语言中默认的 `http.Client` 是没有设置超时时间的。

因此就会导致一直阻塞，一直阻塞就一直爽，Goroutine 自然也就持续暴涨，不断泄露，最终占满资源，导致事故。

在 Go 工程中，我们一般建议至少对 `http.Client` 设置超时时间：

```golang
    httpClient := http.Client{
        Timeout: time.Second * 15,
    }
```
并且要做限流、熔断等措施，以防突发流量造成依赖崩塌，依然吃 P0。

### 互斥锁忘记解锁

第五个例子：

```golang
func main() {
    total := 0
    defer func() {
        time.Sleep(time.Second)
        fmt.Println("total: ", total)
        fmt.Println("goroutines: ", runtime.NumGoroutine())
	}()

    var mutex sync.Mutex
    for i := 0; i < 10; i++ {
        go func() {
            mutex.Lock()
            total += 1
        }()
    }
}
```

输出结果：

```
total:  1
goroutines:  10
```

在这个例子中，第一个互斥锁 `sync.Mutex` 加锁了，但是他可能在处理业务逻辑，又或是忘记 `Unlock` 了。

因此导致后面的所有 `sync.Mutex` 想加锁，却因未释放又都阻塞住了。一般在 Go 工程中，我们建议如下写法：

```golang
    var mutex sync.Mutex
    for i := 0; i < 10; i++ {
        go func() {
            mutex.Lock()
            defer mutex.Unlock()
            total += 1
    }()
    }
```

### 同步锁使用不当

第六个例子：

```golang
func handle(v int) {
    var wg sync.WaitGroup
    wg.Add(5)
    for i := 0; i < v; i++ {
        fmt.Println("脑子进煎鱼了")
        wg.Done()
    }
    wg.Wait()
}

func main() {
    defer func() {
        fmt.Println("goroutines: ", runtime.NumGoroutine())
    }()

    go handle(3)
    time.Sleep(time.Second)
}
```

在这个例子中，我们调用了同步编排 `sync.WaitGroup`，模拟了一遍我们会从外部传入循环遍历的控制变量。

但由于 `wg.Add` 的数量与 `wg.Done` 数量并不匹配，因此在调用 `wg.Wait` 方法后一直阻塞等待。

在 Go 工程中使用，我们会建议如下写法：

```golang
    var wg sync.WaitGroup
    for i := 0; i < v; i++ {
        wg.Add(1)
        defer wg.Done()
        fmt.Println("脑子进煎鱼了")
    }
    wg.Wait()
```

## 排查方法

我们可以调用 `runtime.NumGoroutine` 方法来获取 Goroutine 的运行数量，进行前后一比较，就能知道有没有泄露了。

但在业务服务的运行场景中，Goroutine 内导致的泄露，大多数处于生产、测试环境，因此更多的是使用 PProf：

```golang
import (
    "net/http"
     _ "net/http/pprof"
)

http.ListenAndServe("localhost:6060", nil))
```

只要我们调用 `http://localhost:6060/debug/pprof/goroutine?debug=1`，PProf 会返回所有带有堆栈跟踪的 Goroutine 列表。

也可以利用 PProf 的其他特性进行综合查看和分析，这块参考我之前写的《Go 大杀器之性能剖析 PProf》，基本是全村最全的教程了。

## 总结

在今天这篇文章中，我们针对 Goroutine 泄露的 N 种常见的方式方法进行了一一分析，虽说看起来都是比较基础的场景。

但结合在实际业务代码中，就是一大坨中的某个细节导致全盘皆输了，希望上面几个案例能够给大家带来警惕。

而面试官爱问，怕不是自己踩过许多坑，也希望进来的同僚，也是身经百战了。

靠谱的工程师，而非只是八股工程师。

## 参考

- 波罗学大佬的《[Go 笔记之如何防止 goroutine 泄露](https://zhuanlan.zhihu.com/p/74090074)》
- 二斗斗的《[怎么看待Goroutine 泄露](https://zhuanlan.zhihu.com/p/139689803)》