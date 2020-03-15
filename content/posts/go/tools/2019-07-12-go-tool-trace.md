---

title:      "Go 大杀器之跟踪剖析 trace"
date:       2019-07-12 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
---

![image](https://s2.ax1x.com/2020/02/15/1x1phF.png)

在 Go 中有许许多多的分析工具，在之前我有写过一篇 《Golang 大杀器之性能剖析 PProf》 来介绍 PProf，如果有小伙伴感兴趣可以去我博客看看。

但单单使用 PProf 有时候不一定足够完整，因为在真实的程序中还包含许多的隐藏动作，例如 Goroutine 在执行时会做哪些操作？执行/阻塞了多长时间？在什么时候阻止？在哪里被阻止的？谁又锁/解锁了它们？GC 是怎么影响到 Goroutine 的执行的？这些东西用 PProf 是很难分析出来的，但如果你又想知道上述的答案的话，你可以用本文的主角 `go tool trace` 来打开新世界的大门。目录如下：

![image](https://s2.ax1x.com/2020/02/15/1x1P1J.png)

## 初步了解

```go
import (
	"os"
	"runtime/trace"
)

func main() {
	trace.Start(os.Stderr)
	defer trace.Stop()

	ch := make(chan string)
	go func() {
		ch <- "EDDYCJY"
	}()

	<-ch
}
```

生成跟踪文件：

```
$ go run main.go 2> trace.out
```

启动可视化界面：

```
$ go tool trace trace.out
2019/06/22 16:14:52 Parsing trace...
2019/06/22 16:14:52 Splitting trace...
2019/06/22 16:14:52 Opening browser. Trace viewer is listening on http://127.0.0.1:57321
```

查看可视化界面：

![image](https://s2.ax1x.com/2020/02/15/1x1FXR.png)

- View trace：查看跟踪
- Goroutine analysis：Goroutine 分析
- Network blocking profile：网络阻塞概况
- Synchronization blocking profile：同步阻塞概况
- Syscall blocking profile：系统调用阻塞概况
- Scheduler latency profile：调度延迟概况
- User defined tasks：用户自定义任务
- User defined regions：用户自定义区域
- Minimum mutator utilization：最低 Mutator 利用率

### Scheduler latency profile

在刚开始查看问题时，除非是很明显的现象，否则不应该一开始就陷入细节，因此我们一般先查看 “Scheduler latency profile”，我们能通过 Graph 看到整体的调用开销情况，如下：

![image](https://s2.ax1x.com/2020/02/15/1x1K9e.png)

演示程序比较简单，因此这里就两块，一个是 `trace` 本身，另外一个是 `channel` 的收发。

### Goroutine analysis

第二步看 “Goroutine analysis”，我们能通过这个功能看到整个运行过程中，每个函数块有多少个有 Goroutine 在跑，并且观察每个的 Goroutine 的运行开销都花费在哪个阶段。如下：

![image](https://s2.ax1x.com/2020/02/15/1x1ljA.png)

通过上图我们可以看到共有 3 个 goroutine，分别是 `runtime.main`、`runtime/trace.Start.func1`、`main.main.func1`，那么它都做了些什么事呢，接下来我们可以通过点击具体细项去观察。如下：

![image](https://s2.ax1x.com/2020/02/15/1x18Bt.jpg)

同时也可以看到当前 Goroutine 在整个调用耗时中的占比，以及 GC 清扫和 GC 暂停等待的一些开销。如果你觉得还不够，可以把图表下载下来分析，相当于把整个 Goroutine 运行时掰开来看了，这块能够很好的帮助我们**对 Goroutine 运行阶段做一个的剖析，可以得知到底慢哪，然后再决定下一步的排查方向**。如下：

| 名称                  | 含义         | 耗时   |
| --------------------- | ------------ | ------ |
| Execution Time        | 执行时间     | 3140ns |
| Network Wait Time     | 网络等待时间 | 0ns    |
| Sync Block Time       | 同步阻塞时间 | 0ns    |
| Blocking Syscall Time | 调用阻塞时间 | 0ns    |
| Scheduler Wait Time   | 调度等待时间 | 14ns   |
| GC Sweeping           | GC 清扫      | 0ns    |
| GC Pause              | GC 暂停      | 0ns    |

### View trace

在对当前程序的 Goroutine 运行分布有了初步了解后，我们再通过 “查看跟踪” 看看之间的关联性，如下：

![image](https://s2.ax1x.com/2020/02/15/1x1GHP.png)

这个跟踪图粗略一看，相信有的小伙伴会比较懵逼，我们可以依据注解一块块查看，如下：

1. 时间线：显示执行的时间单元，根据时间维度的不同可以调整区间，具体可执行 `shift` + `?` 查看帮助手册。
2. 堆：显示执行期间的内存分配和释放情况。
3. 协程：显示在执行期间的每个 Goroutine 运行阶段有多少个协程在运行，其包含 GC 等待（GCWaiting）、可运行（Runnable）、运行中（Running）这三种状态。
4. OS 线程：显示在执行期间有多少个线程在运行，其包含正在调用 Syscall（InSyscall）、运行中（Running）这两种状态。
5. 虚拟处理器：每个虚拟处理器显示一行，虚拟处理器的数量一般默认为系统内核数。
6. 协程和事件：显示在每个虚拟处理器上有什么 Goroutine 正在运行，而连线行为代表事件关联。

![image](https://s2.ax1x.com/2020/02/15/1x1YAf.jpg)

点击具体的 Goroutine 行为后可以看到其相关联的详细信息，这块很简单，大家实际操作一下就懂了。文字解释如下：

- Start：开始时间
- Wall Duration：持续时间
- Self Time：执行时间
- Start Stack Trace：开始时的堆栈信息
- End Stack Trace：结束时的堆栈信息
- Incoming flow：输入流
- Outgoing flow：输出流
- Preceding events：之前的事件
- Following events：之后的事件
- All connected：所有连接的事件

### View Events

我们可以通过点击 View Options-Flow events、Following events 等方式，查看我们应用运行中的事件流情况。如下：

![image](https://s2.ax1x.com/2020/02/15/1x1d3Q.png)

通过分析图上的事件流，我们可得知这程序从 `G1 runtime.main` 开始运行，在运行时创建了 2 个 Goroutine，先是创建 `G18 runtime/trace.Start.func1`，然后再是 `G19 main.main.func1` 。而同时我们可以通过其 Goroutine Name 去了解它的调用类型，如：`runtime/trace.Start.func1` 就是程序中在 `main.main` 调用了 `runtime/trace.Start` 方法，然后该方法又利用协程创建了一个闭包 `func1` 去进行调用。

![image](https://s2.ax1x.com/2020/02/15/1x1Dun.png)

在这里我们结合开头的代码去看的话，很明显就是 `ch` 的输入输出的过程了。

## 结合实战

今天生产环境突然出现了问题，机智的你早已埋好 `_ "net/http/pprof"` 这个神奇的工具，你麻利的执行了如下命令：

- curl http://127.0.0.1:6060/debug/pprof/trace\?seconds\=20 > trace.out
- go tool trace trace.out

### View trace

你很快的看到了熟悉的 List 界面，然后不信邪点开了 View trace 界面，如下：

![image](https://s2.ax1x.com/2020/02/15/1x1cNT.jpg)

完全看懵的你，稳住，对着合适的区域执行快捷键 `W` 不断地放大时间线，如下：

![image](https://s2.ax1x.com/2020/02/15/1x1ID1.jpg)

经过初步排查，你发现上述绝大部分的 G 竟然都和 `google.golang.org/grpc.(*Server).Serve.func` 有关，关联的一大串也是 `Serve` 所触发的相关动作。

![image](https://s2.ax1x.com/2020/02/16/3pNw9I.jpg)

这时候有经验的你心里已经有了初步结论，你可以继续追踪 View trace 深入进去，不过我建议先鸟瞰全貌，因此我们再往下看 “Network blocking profile” 和 “Syscall blocking profile” 所提供的信息，如下：

### Network blocking profile

![image](https://s2.ax1x.com/2020/02/16/3pNfCn.jpg)

### Syscall blocking profile

![image](https://s2.ax1x.com/2020/02/16/3pN7bF.jpg)

通过对以上三项的跟踪分析，加上这个泄露，这个阻塞的耗时，这个涉及的内部方法名，很明显就是哪位又忘记关闭客户端连接了，赶紧改改改。

## 总结

通过本文我们习得了 `go tool trace` 的武林秘籍，它能够跟踪捕获各种执行中的事件，例如 Goroutine 的创建/阻塞/解除阻塞，Syscall 的进入/退出/阻止，GC 事件，Heap 的大小改变，Processor 启动/停止等等。

希望你能够用好 Go 的两大杀器 pprof + trace 组合，此乃排查好搭档，谁用谁清楚，即使他并不万能。

## 参考

- https://about.sourcegraph.com/go/an-introduction-to-go-tool-trace-rhys-hiltner
- https://www.itcodemonkey.com/article/5419.html
- https://making.pusher.com/go-tool-trace/
- https://golang.org/cmd/trace/
- https://docs.google.com/document/d/1FP5apqzBgr7ahCCgFO-yoVhk4YZrNIDNf9RybngBc14/pub
- https://godoc.org/runtime/trace