---
title: "会诱发 Goroutine 挂起的 27 个原因"
date: 2021-12-31T12:55:06+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

上个月面向读者的提问，我们针对 goroutine 泄露中都会看到的大头 runtime.gopark 函数进行了学习和了解，输出了 《[Goroutine 一泄露就看到他，这是个什么？](https://mp.weixin.qq.com/s/x6Kzn7VA1wUz7g8txcBX7A)》。

有小伙伴提到，虽然我们知道了 runtime.gopark 函数的缘起和内在，但其实没有提到 **runtime.gopark 的诱发因素，这是我们日常编码中需要关注的**。

今天这篇文章就和大家一起围观 gopark 的 27 个诱发场景。为了方便阅读，我们会根据分类进行说明。

### 第一部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonZero  |   无  |
|   waitReasonGCAssistMarking  |   GC assist marking  |
|   waitReasonIOWait  |   IO wait |

- waitReasonZero：无正式解释，从使用情况来看。主要在 sleep 和 lock 的 2 个场景中使用。
- waitReasonGCAssistMarking：GC 辅助标记阶段会使得阻塞等待。
- waitReasonIOWait：IO 阻塞等待时，例如：网络请求等。

### 第二部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonChanReceiveNilChan  |   chan receive (nil chan)  |
|   waitReasonChanSendNilChan  |   chan send (nil chan)  |

- waitReasonChanReceiveNilChan：对未初始化的 channel 进行读操作。
- waitReasonChanSendNilChan：对未初始化的 channel 进行写操作。

### 第三部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonDumpingHeap  |  dumping heap |
|   waitReasonGarbageCollection  |   garbage collection  |
|   waitReasonGarbageCollectionScan  |   garbage collection scan  |

- waitReasonDumpingHeap：对 Go Heap 堆 dump 时，这个的使用场景仅在 runtime.debug 时，也就是常见的 pprof 这一类采集时阻塞。
- waitReasonGarbageCollection：在垃圾回收时，主要场景是 GC 标记终止（GC Mark Termination）阶段时触发。
- waitReasonGarbageCollectionScan：在垃圾回收扫描时，主要场景是 GC 标记（GC Mark）扫描 Root 阶段时触发。

### 第四部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonPanicWait  |  panicwait |
|   waitReasonSelect  |   select  |
|   waitReasonSelectNoCases  |   select (no cases)  |

- waitReasonPanicWait：在 main goroutine 发生 panic 时，会触发。
- waitReasonSelect：在调用关键字 select 时会触发。
- waitReasonSelectNoCases：在调用关键字 select 时，若一个 case 都没有，会直接触发。

### 第五部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonGCAssistWait  |  GC assist wait |
|   waitReasonGCSweepWait  |   GC sweep wait  |
|   waitReasonGCScavengeWait  |   GC scavenge wait  |

- waitReasonGCAssistWait：GC 辅助标记阶段中的结束行为，会触发。
- waitReasonGCSweepWait：GC 清扫阶段中的结束行为，会触发。
- waitReasonGCScavengeWait：GC scavenge 阶段的结束行为，会触发。GC Scavenge 主要是新空间的垃圾回收，是一种经常运行、快速的 GC，负责从新空间中清理较小的对象。

### 第六部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonChanReceive  |  chan receive |
|   waitReasonChanSend  |   chan send  |
|   waitReasonFinalizerWait  |   finalizer wait  |

- waitReasonChanReceive：在 channel 进行读操作，会触发。
- waitReasonChanSend：在 channel 进行写操作，会触发。
- waitReasonFinalizerWait：在 finalizer 结束的阶段，会触发。在 Go 程序中，可以通过调用 `runtime.SetFinalizer` 函数来为一个对象设置一个终结者函数。这个行为对应着结束阶段造成的回收。

### 第七部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonForceGCIdle  |  force gc (idle) |
|   waitReasonSemacquire  |   semacquire  |
|   waitReasonSleep  |   sleep  |

- waitReasonForceGCIdle：强制 GC（空闲时间）结束时，会触发。
- waitReasonSemacquire：信号量处理结束时，会触发。
- waitReasonSleep：经典的 sleep 行为，会触发。

### 第八部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonSyncCondWait  |  sync.Cond.Wait |
|   waitReasonTimerGoroutineIdle  |   timer goroutine (idle)  |
|   waitReasonTraceReaderBlocked  |   trace reader (blocked)  |

- waitReasonSyncCondWait：结合 `sync.Cond` 用法能知道，是在调用 `sync.Wait` 方法时所触发。
- waitReasonTimerGoroutineIdle：与 Timer 相关，在没有定时器需要执行任务时，会触发。
- waitReasonTraceReaderBlocked：与 Trace 相关，ReadTrace会返回二进制跟踪数据，将会阻塞直到数据可用。

### 第九部分

|  标识   |  含义   |
| --- | --- |
|   waitReasonWaitForGCCycle  |  wait for GC cycle |
|   waitReasonGCWorkerIdle  |   GC worker (idle)  |
|   waitReasonPreempted  |   preempted  |
|   waitReasonDebugCall  |   debug call  |

- waitReasonWaitForGCCycle：等待 GC 周期，会休眠造成阻塞。
- waitReasonGCWorkerIdle：GC Worker 空闲时，会休眠造成阻塞。
- waitReasonPreempted：发生循环调用抢占时，会会休眠等待调度。
- waitReasonDebugCall：调用 GODEBUG 时，会触发。

## 总结

今天这篇文章是对开头 runtime.gopark 函数的详解文章的一个补充，我们能够对此了解到其诱发的因素。

主要场景为：
1. 通道（Channel）。
2. 垃圾回收（GC）。
3. 休眠（Sleep）。
4. 锁等待（Lock）。
5. 抢占（Preempted）。
6. IO 阻塞（IO Wait）
7. 其他，例如：panic、finalizer、select 等。

我们可以根据这些特性，去拆解可能会造成阻塞的原因。其实也就没必要记了，他们会导致阻塞肯定是由于存在影响控制流的因素，才会导致 gopark 的调用。

活学活用：）