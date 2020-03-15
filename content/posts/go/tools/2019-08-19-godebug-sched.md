---

title:      "用 GODEBUG 看调度跟踪"
date:       2019-08-19 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
---

![image](https://image.eddycjy.com/b01c2ce25e34f80d499f0488d034b00b.png)

让 Go 更强大的原因之一莫过于它的 GODEBUG 工具，GODEBUG 的设置可以让 Go 程序在运行时输出调试信息，可以根据你的要求很直观的看到你想要的调度器或垃圾回收等详细信息，并且还不需要加装其它的插件，非常方便，今天我们将先讲解 GODEBUG 的调度器相关内容，希望对你有所帮助。

不过在开始前，没接触过的小伙伴得先补补如下前置知识，便于更好的了解调试器输出的信息内容。

## 前置知识

Go scheduler 的主要功能是针对在处理器上运行的 OS 线程分发可运行的 Goroutine，而我们一提到调度器，就离不开三个经常被提到的缩写，分别是：

- G：Goroutine，实际上我们每次调用 `go func` 就是生成了一个 G。
- P：处理器，一般为处理器的核数，可以通过 `GOMAXPROCS` 进行修改。
- M：OS 线程

这三者交互实际来源于 Go 的 M: N 调度模型，也就是 M 必须与 P 进行绑定，然后不断地在 M 上循环寻找可运行的 G 来执行相应的任务，如果想具体了解可以详细阅读 [《Go Runtime Scheduler》](https://speakerdeck.com/retervision/go-runtime-scheduler)，我们抽其中的工作流程图进行简单分析，如下:

![image](https://image.eddycjy.com/fb4c6c92c93af3bc2dfc4f13dc167cdf.png)

1. 当我们执行 `go func()` 时，实际上就是创建一个全新的 Goroutine，我们称它为 G。
2. 新创建的 G 会被放入 P 的本地队列（Local Queue）或全局队列（Global Queue）中，准备下一步的动作。
3. 唤醒或创建 M 以便执行 G。
4. 不断地进行事件循环
5. 寻找在可用状态下的 G 进行执行任务
6. 清除后，重新进入事件循环

而在描述中有提到全局和本地这两类队列，其实在功能上来讲都是用于存放正在等待运行的 G，但是不同点在于，本地队列有数量限制，不允许超过 256 个。并且在新建 G 时，会优先选择 P 的本地队列，如果本地队列满了，则将 P 的本地队列的一半的 G 移动到全局队列，这其实可以理解为调度资源的共享和再平衡。

另外我们可以看到图上有 steal 行为，这是用来做什么的呢，我们都知道当你创建新的 G 或者 G 变成可运行状态时，它会被推送加入到当前 P 的本地队列中。但其实当 P 执行 G 完毕后，它也会 “干活”，它会将其从本地队列中弹出 G，同时会检查当前本地队列是否为空，如果为空会随机的从其他 P 的本地队列中尝试窃取一半可运行的 G 到自己的名下。例子如下：

![image](https://image.eddycjy.com/e7ca8f212466d8c15ec0f60b69a1ce4d.png)

在这个例子中，P2 在本地队列中找不到可以运行的 G，它会执行 `work-stealing` 调度算法，随机选择其它的处理器 P1，并从 P1 的本地队列中窃取了三个 G 到它自己的本地队列中去。至此，P1、P2 都拥有了可运行的 G，P1 多余的 G 也不会被浪费，调度资源将会更加平均的在多个处理器中流转。

## GODEBUG

GODEBUG 变量可以控制运行时内的调试变量，参数以逗号分隔，格式为：`name=val`。本文着重点在调度器观察上，将会使用如下两个参数：

- schedtrace：设置 `schedtrace=X` 参数可以使运行时在每 X 毫秒发出一行调度器的摘要信息到标准 err 输出中。
- scheddetail：设置 `schedtrace=X` 和 `scheddetail=1` 可以使运行时在每 X 毫秒发出一次详细的多行信息，信息内容主要包括调度程序、处理器、OS 线程 和 Goroutine 的状态。

### 演示代码

```
func main() {
	wg := sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func(wg *sync.WaitGroup) {
			var counter int
			for i := 0; i < 1e10; i++ {
				counter++
			}
			wg.Done()
		}(&wg)
	}

	wg.Wait()
}
```

### schedtrace

```
$ GODEBUG=schedtrace=1000 ./awesomeProject 
SCHED 0ms: gomaxprocs=4 idleprocs=1 threads=5 spinningthreads=1 idlethreads=0 runqueue=0 [0 0 0 0]
SCHED 1000ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
SCHED 2000ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
SCHED 3001ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
SCHED 4010ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
SCHED 5011ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
SCHED 6012ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 [1 2 2 1]
SCHED 7021ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=4 [0 1 1 0]
SCHED 8023ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=4 [0 1 1 0]
SCHED 9031ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=4 [0 1 1 0]
SCHED 10033ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=4 [0 1 1 0]
SCHED 11038ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=4 [0 1 1 0]
SCHED 12044ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=4 [0 1 1 0]
SCHED 13051ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=4 [0 1 1 0]
SCHED 14052ms: gomaxprocs=4 idleprocs=2 threads=5 
...
```
- sched：每一行都代表调度器的调试信息，后面提示的毫秒数表示启动到现在的运行时间，输出的时间间隔受 `schedtrace` 的值影响。
- gomaxprocs：当前的 CPU 核心数（GOMAXPROCS 的当前值）。
- idleprocs：空闲的处理器数量，后面的数字表示当前的空闲数量。
- threads：OS 线程数量，后面的数字表示当前正在运行的线程数量。
- spinningthreads：自旋状态的 OS 线程数量。
- idlethreads：空闲的线程数量。
- runqueue：全局队列中中的 Goroutine 数量，而后面的 [0 0 1 1] 则分别代表这 4 个 P 的本地队列正在运行的 Goroutine 数量。

在上面我们有提到 “自旋线程” 这个概念，如果你之前没有了解过相关概念，一听 “自旋” 肯定会比较懵，我们引用 《Head First of Golang Scheduler》 的内容来说明：
 
 > 自旋线程的这个说法，是因为 Go Scheduler 的设计者在考虑了 “OS 的资源利用率” 以及 “频繁的线程抢占给 OS 带来的负载” 之后，提出了 “Spinning Thread” 的概念。也就是当 “自旋线程” 没有找到可供其调度执行的 Goroutine 时，并不会销毁该线程 ，而是采取 “自旋” 的操作保存了下来。虽然看起来这是浪费了一些资源，但是考虑一下 syscall 的情景就可以知道，比起 “自旋"，线程间频繁的抢占以及频繁的创建和销毁操作可能带来的危害会更大。

### scheddetail

如果我们想要更详细的看到调度器的完整信息时，我们可以增加 `scheddetail` 参数，就能够更进一步的查看调度的细节逻辑，如下：

```
$ GODEBUG=scheddetail=1,schedtrace=1000 ./awesomeProject
SCHED 1000ms: gomaxprocs=4 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0
  P0: status=1 schedtick=2 syscalltick=0 m=3 runqsize=3 gfreecnt=0
  P1: status=1 schedtick=2 syscalltick=0 m=4 runqsize=1 gfreecnt=0
  P2: status=1 schedtick=2 syscalltick=0 m=0 runqsize=1 gfreecnt=0
  P3: status=1 schedtick=1 syscalltick=0 m=2 runqsize=1 gfreecnt=0
  M4: p=1 curg=18 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 spinning=false blocked=false lockedg=-1
  M3: p=0 curg=22 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 spinning=false blocked=false lockedg=-1
  M2: p=3 curg=24 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 spinning=false blocked=false lockedg=-1
  M1: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 spinning=false blocked=false lockedg=-1
  M0: p=2 curg=26 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 spinning=false blocked=false lockedg=-1
  G1: status=4(semacquire) m=-1 lockedm=-1
  G2: status=4(force gc (idle)) m=-1 lockedm=-1
  G3: status=4(GC sweep wait) m=-1 lockedm=-1
  G17: status=1() m=-1 lockedm=-1
  G18: status=2() m=4 lockedm=-1
  G19: status=1() m=-1 lockedm=-1
  G20: status=1() m=-1 lockedm=-1
  G21: status=1() m=-1 lockedm=-1
  G22: status=2() m=3 lockedm=-1
  G23: status=1() m=-1 lockedm=-1
  G24: status=2() m=2 lockedm=-1
  G25: status=1() m=-1 lockedm=-1
  G26: status=2() m=0 lockedm=-1
```

在这里我们抽取了 1000ms 时的调试信息来查看，信息量比较大，我们先从每一个字段开始了解。如下：

#### G

- status：G 的运行状态。
- m：隶属哪一个 M。
- lockedm：是否有锁定 M。

在第一点中我们有提到 G 的运行状态，这对于分析内部流转非常的有用，共涉及如下 9 种状态：

状态 | 值 | 含义
---|---| ---
_Gidle | 0 | 刚刚被分配，还没有进行初始化。
_Grunnable | 1 | 已经在运行队列中，还没有执行用户代码。
_Grunning | 2 | 不在运行队列里中，已经可以执行用户代码，此时已经分配了 M 和 P。
_Gsyscall | 3 | 正在执行系统调用，此时分配了 M。
_Gwaiting | 4 | 在运行时被阻止，没有执行用户代码，也不在运行队列中，此时它正在某处阻塞等待中。
_Gmoribund_unused | 5 | 尚未使用，但是在 gdb 中进行了硬编码。
_Gdead | 6 | 尚未使用，这个状态可能是刚退出或是刚被初始化，此时它并没有执行用户代码，有可能有也有可能没有分配堆栈。
_Genqueue_unused | 7 | 尚未使用。
_Gcopystack | 8 | 正在复制堆栈，并没有执行用户代码，也不在运行队列中。

在理解了各类的状态的意思后，我们结合上述案例看看，如下：

```
G1: status=4(semacquire) m=-1 lockedm=-1
G2: status=4(force gc (idle)) m=-1 lockedm=-1
G3: status=4(GC sweep wait) m=-1 lockedm=-1
G17: status=1() m=-1 lockedm=-1
G18: status=2() m=4 lockedm=-1
```

在这个片段中，G1 的运行状态为 `_Gwaiting`，并没有分配 M 和锁定。这时候你可能好奇在片段中括号里的是什么东西呢，其实是因为该 `status=4` 是表示 `Goroutine` 在**运行时时被阻止**，而阻止它的事件就是 `semacquire` 事件，是因为 `semacquire` 会检查信号量的情况，在合适的时机就调用 `goparkunlock` 函数，把当前 `Goroutine` 放进等待队列，并把它设为 `_Gwaiting` 状态。

那么在实际运行中还有什么原因会导致这种现象呢，我们一起看看，如下：

```
	waitReasonZero                                    // ""
	waitReasonGCAssistMarking                         // "GC assist marking"
	waitReasonIOWait                                  // "IO wait"
	waitReasonChanReceiveNilChan                      // "chan receive (nil chan)"
	waitReasonChanSendNilChan                         // "chan send (nil chan)"
	waitReasonDumpingHeap                             // "dumping heap"
	waitReasonGarbageCollection                       // "garbage collection"
	waitReasonGarbageCollectionScan                   // "garbage collection scan"
	waitReasonPanicWait                               // "panicwait"
	waitReasonSelect                                  // "select"
	waitReasonSelectNoCases                           // "select (no cases)"
	waitReasonGCAssistWait                            // "GC assist wait"
	waitReasonGCSweepWait                             // "GC sweep wait"
	waitReasonChanReceive                             // "chan receive"
	waitReasonChanSend                                // "chan send"
	waitReasonFinalizerWait                           // "finalizer wait"
	waitReasonForceGGIdle                             // "force gc (idle)"
	waitReasonSemacquire                              // "semacquire"
	waitReasonSleep                                   // "sleep"
	waitReasonSyncCondWait                            // "sync.Cond.Wait"
	waitReasonTimerGoroutineIdle                      // "timer goroutine (idle)"
	waitReasonTraceReaderBlocked                      // "trace reader (blocked)"
	waitReasonWaitForGCCycle                          // "wait for GC cycle"
	waitReasonGCWorkerIdle                            // "GC worker (idle)"
```

我们通过以上 `waitReason` 可以了解到 `Goroutine` 会被暂停运行的原因要素，也就是会出现在括号中的事件。

#### M

- p：隶属哪一个 P。
- curg：当前正在使用哪个 G。
- runqsize：运行队列中的 G 数量。
- gfreecnt：可用的G（状态为 Gdead）。
- mallocing：是否正在分配内存。
- throwing：是否抛出异常。
- preemptoff：不等于空字符串的话，保持 curg 在这个 m 上运行。

#### P

- status：P 的运行状态。
- schedtick：P 的调度次数。
- syscalltick：P 的系统调用次数。
- m：隶属哪一个 M。
- runqsize：运行队列中的 G 数量。
- gfreecnt：可用的G（状态为 Gdead）。

状态 | 值 | 含义
---|---| ---
_Pidle | 0 | 刚刚被分配，还没有进行进行初始化。
_Prunning | 1 | 当 M 与 P 绑定调用 acquirep 时，P 的状态会改变为 _Prunning。
_Psyscall | 2 | 正在执行系统调用。
_Pgcstop | 3 | 暂停运行，此时系统正在进行 GC，直至 GC 结束后才会转变到下一个状态阶段。
_Pdead | 4 | 废弃，不再使用。

## 总结

通过本文我们学习到了调度的一些基础知识，再通过神奇的 GODEBUG 掌握了观察调度器的方式方法，你想想，是不是可以和我上一篇文章的 `go tool trace` 来结合使用呢，在实际的使用中，类似的办法有很多，组合巧用是重点。

## 参考

- [Debugging performance issues in Go programs](https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs)
- [A whirlwind tour of Go’s runtime environment variables](https://dave.cheney.net/tag/godebug)
- [Go调度器系列（2）宏观看调度器](https://mp.weixin.qq.com/s?__biz=Mzg3MTA0NDQ1OQ==&mid=2247483907&idx=2&sn=c955372683bc0078e14227702ab0a35e&chksm=ce85c607f9f24f116158043f63f7ca11dc88cd519393ba182261f0d7fc328c7b6a94fef4e416&scene=38#wechat_redirect)
- [Go's work-stealing scheduler](https://rakyll.org/scheduler/)
- [Scheduler Tracing In Go](https://www.ardanlabs.com/blog/2015/02/scheduler-tracing-in-go.html)
- [Head First of Golang Scheduler](https://zhuanlan.zhihu.com/p/42057783)
- [goroutine 的状态切换](http://xargin.com/state-of-goroutine/)
- [Environment_Variables](https://golang.org/pkg/runtime/#hdr-Environment_Variables)