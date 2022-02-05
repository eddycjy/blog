---
title: "Go 什么时候会触发 GC？"
date: 2021-12-31T12:55:08+08:00
toc: true
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

Go 语言作为一门新语言，在早期经常遭到唾弃的就是在垃圾回收（下称：GC）机制中 STW（Stop-The-World）的时间过长。

那么这个时候，我们又会好奇一点，作为 STW 的起始，Go 语言中什么时候才会触发 GC 呢？

今天就由煎鱼带大家一起来学习研讨一轮。

## 什么是 GC

在计算机科学中，垃圾回收（GC）是一种自动管理内存的机制，垃圾回收器会去尝试回收程序不再使用的对象及其占用的内存。

最早 John McCarthy 在 1959 年左右发明了垃圾回收，以简化 Lisp 中的手动内存管理的机制（来自 @wikipedia）。

![图来自网络](https://files.mdnice.com/user/3610/09425faf-f521-43a9-a7fa-1db3c5163914.png)

## 为什么要 GC

手动管理内存挺麻烦，管错或者管漏内存也很糟糕，将会直接导致程序不稳定（持续泄露）甚至直接崩溃。

## GC 触发场景

GC 触发的场景主要分为两大类，分别是：
1. 系统触发：运行时自行根据内置的条件，检查、发现到，则进行 GC 处理，维护整个应用程序的可用性。
2. 手动触发：开发者在业务代码中自行调用 ` runtime.GC` 方法来触发 GC 行为。

### 系统触发

在系统触发的场景中，Go 源码的 `src/runtime/mgc.go` 文件，明确标识了 GC 系统触发的三种场景，分别如下：

```golang
const (
	gcTriggerHeap gcTriggerKind = iota
	gcTriggerTime
	gcTriggerCycle
)
```
- gcTriggerHeap：当所分配的堆大小达到阈值（由控制器计算的触发堆的大小）时，将会触发。
- gcTriggerTime：当距离上一个 GC 周期的时间超过一定时间时，将会触发。
    -时间周期以 `runtime.forcegcperiod` 变量为准，默认 2 分钟。
- gcTriggerCycle：如果没有开启 GC，则启动 GC。
    - 在手动触发的 `runtime.GC` 方法中涉及。

### 手动触发

在手动触发的场景下，Go 语言中仅有 `runtime.GC` 方法可以触发，也就没什么额外的分类的。

![](https://files.mdnice.com/user/3610/8da69a96-6445-4e20-9234-a0c74ba2b5de.png)

但我们要思考的是，一般我们在什么业务场景中，要涉及到手动干涉 GC，强制触发他呢？

需要手动强制触发的场景极其少见，可能会是在某些业务方法执行完后，因其占用了过多的内存，需要人为释放。又或是 debug 程序所需。

## 基本流程

在了解到 Go 语言会触发 GC 的场景后，我们进一步看看触发 GC 的流程代码是怎么样的，我们可以借助手动触发的 `runtime.GC` 方法来作为突破口。

核心代码如下：

```golang
func GC() {
	n := atomic.Load(&work.cycles)
	gcWaitOnMark(n)

	gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})
  
	gcWaitOnMark(n + 1)

	for atomic.Load(&work.cycles) == n+1 && sweepone() != ^uintptr(0) {
		sweep.nbgsweep++
		Gosched()
	}
  
	for atomic.Load(&work.cycles) == n+1 && atomic.Load(&mheap_.sweepers) != 0 {
		Gosched()
	}
  
	mp := acquirem()
	cycle := atomic.Load(&work.cycles)
	if cycle == n+1 || (gcphase == _GCmark && cycle == n+2) {
		mProf_PostSweep()
	}
	releasem(mp)
}
```
1. 在开始新的一轮 GC 周期前，需要调用 `gcWaitOnMark` 方法上一轮 GC 的标记结束（含扫描终止、标记、或标记终止等）。
2. 开始新的一轮 GC 周期，调用 `gcStart` 方法触发 GC 行为，开始扫描标记阶段。
3. 需要调用 `gcWaitOnMark` 方法等待，直到当前 GC 周期的扫描、标记、标记终止完成。
4. 需要调用 `sweepone` 方法，扫描未扫除的堆跨度，并持续扫除，保证清理完成。在等待扫除完毕前的阻塞时间，会调用 `Gosched` 让出。
5. 在本轮 GC 已经基本完成后，会调用 `mProf_PostSweep` 方法。以此记录最后一次标记终止时的堆配置文件快照。
6. 结束，释放 M。

## 在哪触发

看完 GC 的基本流程后，我们有了一个基本的了解。但可能又有小伙伴有疑惑了？

本文的标题是 “GC 什么时候会触发 GC”，虽然我们前面知道了触发的时机。但是....Go 是哪里实现的触发的机制，似乎在流程中完全没有看到？

### 监控线程

实质上在 Go 运行时（runtime）初始化时，会启动一个 goroutine，用于处理 GC 机制的相关事项。

代码如下：

```golang
func init() {
	go forcegchelper()
}

func forcegchelper() {
	forcegc.g = getg()
	lockInit(&forcegc.lock, lockRankForcegc)
	for {
		lock(&forcegc.lock)
		if forcegc.idle != 0 {
			throw("forcegc: phase error")
		}
		atomic.Store(&forcegc.idle, 1)
		goparkunlock(&forcegc.lock, waitReasonForceGCIdle, traceEvGoBlock, 1)
    // this goroutine is explicitly resumed by sysmon
		if debug.gctrace > 0 {
			println("GC forced")
		}

		gcStart(gcTrigger{kind: gcTriggerTime, now: nanotime()})
	}
}
```

在这段程序中，需要特别关注的是在 `forcegchelper` 方法中，会调用 `goparkunlock` 方法让该 goroutine 陷入休眠等待状态，以减少不必要的资源开销。

在休眠后，会由 `sysmon` 这一个系统监控线程来进行监控、唤醒等行为：

```golang
func sysmon() {
	...
	for {
		...
		// check if we need to force a GC
		if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
			lock(&forcegc.lock)
			forcegc.idle = 0
			var list gList
			list.push(forcegc.g)
			injectglist(&list)
			unlock(&forcegc.lock)
		}
		if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
			lasttrace = now
			schedtrace(debug.scheddetail > 0)
		}
		unlock(&sched.sysmonlock)
	}
}
```

这段代码核心的行为就是不断地在 for 循环中，对 `gcTriggerTime` 和 `now` 变量进行比较，判断是否达到一定的时间（默认为 2 分钟）。

若达到意味着满足条件，会将 `forcegc.g` 放到全局队列中接受新的一轮调度，再进行对上面 `forcegchelper` 的唤醒。

### 堆内存申请

在了解定时触发的机制后，另外一个场景就是分配的堆空间的时候，那么我们要看的地方就非常明确了。

那就是运行时申请堆内存的 `mallocgc` 方法。核心代码如下：

```golang
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	shouldhelpgc := false
	...
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			...
			// Allocate a new maxTinySize block.
			span = c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			...
			spc := makeSpanClass(sizeclass, noscan)
			span = c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			...
		}
	} else {
		shouldhelpgc = true
		span = c.allocLarge(size, needzero, noscan)
		...
	}

	if shouldhelpgc {
		if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
			gcStart(t)
		}
	}

	return x
}
```

- 小对象：如果申请小对象时，发现当前内存空间不存在空闲跨度时，将会需要调用 `nextFree` 方法获取新的可用的对象，可能会触发 GC 行为。
- 大对象：如果申请大于 32k 以上的大对象时，可能会触发 GC 行为。

## 总结

在这篇文章中，我们介绍了 Go 语言触发 GC 的两大类场景，并分别基于大类中的细分场景进行了一一说明。

一般来讲，我们对其了解大概就可以了。若小伙伴们对其内部具体实现感兴趣，也可以以文章中的代码具体再打开看。

但需要注意，很有可能 Go 版本一升级，可能又变了，学思想要紧 ：）
