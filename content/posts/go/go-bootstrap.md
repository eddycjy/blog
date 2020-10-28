---
title: "Go 应用程序是怎么运行起来的？"
date: 2020-10-08T15:57:18+08:00
toc: true
images:
tags: 
  - go
---

自古应用程序均从 Hello World 开始，你我所写的 Go 语言亦然：

```
import "fmt"

func main() {
	fmt.Println("hello world.")
}
```

这段程序的输出结果为 `hello world.`，就是这么的简单又直接。但这时候又不禁思考了起来，这个 `hello world.` 是怎么输出来，经历了什么过程。

真是非常的好奇，今天我们就通过这篇文章来一探究竟。

## 引导阶段

### 查找入口

开始剖析之路，首先编译上文提到的示例程序：

```shell
$ GOFLAGS="-ldflags=-compressdwarf=false" go build 
```

在命令中指定了 GOFLAGS 参数，这是因为在 Go1.11 起，为了减少二进制文件大小，调试信息会被压缩。导致在 MacOS 上使用 gdb 时无法理解压缩的 DWARF 的含义是什么（而我恰恰就是用的 MacOS）。

因此需要在本次调试中将其关闭，再使用 gdb 进行调试，以此达到观察的目的：

```shell
$ gdb awesomeProject 
(gdb) info files
Symbols from "/Users/eddycjy/go-application/awesomeProject/awesomeProject".
Local exec file:
	`/Users/eddycjy/go-application/awesomeProject/awesomeProject', file type mach-o-x86-64.
	Entry point: 0x1063c80
	0x0000000001001000 - 0x00000000010a6aca is .text
	...
(gdb) b *0x1063c80
Breakpoint 1 at 0x1063c80: file /usr/local/Cellar/go/1.15/libexec/src/runtime/rt0_darwin_amd64.s, line 8.
```

通过 Entry point 的调试，可看到真正的程序入口在 runtime 包中，不同的计算机架构指向不同，例如：MacOS 在 `src/runtime/rt0_darwin_amd64.s`，Linux 在 `src/runtime/rt0_linux_amd64.s`。

```
Breakpoint 1 at 0x1063c80: file /usr/local/Cellar/go/1.15/libexec/src/runtime/rt0_darwin_amd64.s, line 8.
```

其最终指向了 rt0_darwin_amd64.s 文件，这个文件名称非常的直观，rt0 代表 runtime0 的缩写，指代运行时的创世，超级奶爸；darwin 代表目标操作系统（GOOS），amd64 代表目标操作系统架构（GOHOSTARCH）。

同时 Go 语言还支持更多的目标系统架构，例如：AMD64、AMR、MIPS、WASM 等：

![image](https://image.eddycjy.com/981720dfbce750bec26fc394e97d9ff7.jpg)

若有兴趣可到 `src/runtime` 目录下进一步查看。

### 入口方法

在 rt0_linux_amd64.s 文件中，可发现 `_rt0_amd64_darwin` JMP 跳转到了 `_rt0_amd64` 方法：

```
TEXT _rt0_amd64_darwin(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB)
...
```

紧接着又跳转到 `runtime·rt0_go` 方法：

```
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB)
```

该方法将程序输入的 argc 和 argv 从内存移动到寄存器中。栈指针（SP）的前两个值分别是 argc 和 argv，其对应参数的数量和具体各参数的值。

### 开启主线

程序参数准备就绪后，正式初始化的方法落在 `runtime·rt0_go` 方法中：

```
TEXT runtime·rt0_go(SB),NOSPLIT,$0
	...
	CALL	runtime·check(SB)
	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)
	CALL	runtime·osinit(SB)
	CALL	runtime·schedinit(SB)

	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX

	// start this M
	CALL	runtime·mstart(SB)
	...
```


- runtime.check：运行时类型检查，主要是校验编译器的翻译工作是否正确，是否有 “坑”。基本代码均为检查 `int8` 在 `unsafe.Sizeof` 方法下是否等于 1 这类动作。

- runtime.args：系统参数传递，主要是将系统参数转换传递给程序使用。

- runtime.osinit：系统基本参数设置，主要是获取 CPU 核心数和内存物理页大小。

- runtime.schedinit：进行各种运行时组件的初始化，包含调度器、内存分配器、堆、栈、GC 等一大堆初始化工作。是后续的重点关注对象。

- runtime·main：主要工作是运行 main goroutine，虽然在`runtime·rt0_go` 中指向的是`$runtime·mainPC` ，但实质指向的是 `runtime.main`。

- runtime.newproc：创建一个新的 goroutine 将其放入 g 的等待运行队列中去。且绑定 `runtime.main` 方法，也就是应用程序中的入口 main 方法。

- runtime.mstart：调度器开始进行循环调度。

在 `runtime·rt0_go` 方法中，其主要是完成各类运行时的检查，系统参数设置和获取，并进行大量的 Go 基础组件初始化。初始化完毕后进行 main goroutine 的运行，并放入等待队列（GMP），最后调度器开始进行循环调度。

## 总结

根据上述源码剖析，可以得出如下 Go 应用程序引导的流程图：

![image](https://image.eddycjy.com/057c1ccb06c16e8c5f38ff5800e3fa63.jpg)

在 Go 语言中，实际的运行入口并不是用户日常所写的 `main func`，更不是 `runtime.main` 方法，而是从 `rt0_*_amd64.s` 开始，最终再一路 JMP 到 `runtime·rt0_go` 里去，再在该方法里完成一系列 Go 自身所需要完成的绝大部分初始化动作。

其中包括运行时类型检查、系统参数传递、CPU 核数获取及设置、运行时组件的初始化（调度器、内存分配器、堆、栈、GC 等）、运行 main goroutine 和相应的 GMP 等大量缺省行为，还会涉及到调度器相关的大量知识。

后续将会继续剖析将进一步剖析 `runtime·rt0_go` 里的爱与恨，尤其像是 `runtime.main`、`runtime.schedinit` 等调度方法，都有非常大的学习价值，有兴趣的小伙伴可以持续关注。
