---
title: "Goroutine 一泄露就看到他，这是个什么？"
date: 2021-12-31T12:55:01+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

作为一个 Go 语言的使用大户，常常就有人冷不丁的，一下就泄露了...泄露了啥？

表象来看当然是 goroutine 泄露了，这时候就会有小伙伴开始跑去拉取 PProf。就会看到类似下面这张图：

![](https://files.mdnice.com/user/3610/78c8dadf-4f5c-4dfe-be36-d39bfa9d13d3.png)

重点会看到 `runtime.gopark` 这个函数，在所有的 goroutine 泄露中都会看到有，并且都会是大头。

既然是大头，也就会有许多朋友以为他是泄漏点，在那一顿猛查，那这个函数到底是什么，作用是？ 

## runtime.gopark 是何物

想要知道 `runtime.gopark` 函数是作用，最快的办法就是看源码了。其实现细节在 src/runtime/proc.go 文件中。

源代码如下：

```golang
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	
	mcall(park_m)
}
```

该函数主要作用有三大点：
- 调用 `acquirem` 函数：
    - 获取当前 goroutine 所绑定的 m，设置各类所需数据。
    - 调用 `releasem` 函数**将当前 goroutine 和其 m 的绑定关系解除**。
- 调用 `park_m` 函数：
    - 将当前 goroutine 的状态从 `_Grunning` 切换为 `_Gwaiting`，也就是等待状态。
    - 删除 m 和当前 goroutine m->curg（简称gp）之间的关联。
- 调用 `mcall` 函数，仅会在需要进行 goroutiine 切换时会被调用：
    - 切换当前线程的堆栈，从 g 的堆栈切换到 g0 的堆栈并调用 fn(g) 函数。
    - 将 g 的当前 PC/SP 保存在 g->sched 中，以便后续调用 goready 函数时可以恢复运行现场。

熟读了其源码后，我们可得知该函数的关键作用就是**将当前的 goroutine 放入等待状态**，这意味着 goroutine 被暂时被搁置了，也就是被运行时调度器暂停了。

## 缘由

回到最初的问题，之所以 goroutine 泄露，你就会看到大量的 `runtime.gopark` 函数，这是因为 goroutine 泄露一般不会单单只是一个 goroutine，肯定是会有多个的。

同时这些 goroutine 在调用了 `runtime.gopark` 函数后都被暂停了，也就是进入休眠状态，自然而然也就停留在此。

直至满足条件后再被 `runtime.goready` 函数唤醒，该函数会将已准备就绪的 goroutine 切换状态，再加入运行队列，等待调度器的新一轮调度。

## 思考

前几天就有读者在我的 Go 读者群中咨询了下述问题，也和  `runtime.gopark` 函数有关。问题如下：

![](https://files.mdnice.com/user/3610/eb5d0483-8193-4adf-881e-490825d137fe.png)

经过上述的分析，显然 `runtime.gopark` 不是 goroutine 的一种状态，导致 goroutine 状态变更只是他的执行过程中所涉及到，产生的一个结果。

而 goroutine 的状态一共有 9 种，有兴趣的小伙伴可以了解。如下：

状态  | 含义
---|---| ---
_Gidle  | 刚刚被分配，还没有进行初始化。
_Grunnable  | 已经在运行队列中，还没有执行用户代码。
_Grunning | 不在运行队列里中，已经可以执行用户代码，此时已经分配了 M 和 P。
_Gsyscall | 正在执行系统调用，此时分配了 M。
_Gwaiting | 在运行时被阻止，没有执行用户代码，也不在运行队列中，此时它正在某处阻塞等待中。
_Gmoribund_unused | 尚未使用，但是在 gdb 中进行了硬编码。
_Gdead | 尚未使用，这个状态可能是刚退出或是刚被初始化，此时它并没有执行用户代码，有可能有也有可能没有分配堆栈。
_Genqueue_unused | 尚未使用。
_Gcopystack | 正在复制堆栈，并没有执行用户代码，也不在运行队列中。

## 总结

在今天这篇文章中，我们介绍了大家最常碰到的 goroutine 泄露，而在泄露后最关心的 `runtime.gopark` 函数的意义，我们从源码再到作用进行了一轮剖析。

下次如果再有人问你 `runtime.gopark` 是干嘛用的，就可以愉快的把这篇文章甩给他，分享你的知识啦 ：）