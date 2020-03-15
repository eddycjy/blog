---

title:      "深入理解 Go panic and recover"
date:       2019-05-21 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - 源码分析
---

作为一个 gophper，我相信你对于 `panic` 和 `recover` 肯定不陌生，但是你有没有想过。当我们执行了这两条语句之后。底层到底发生了什么事呢？前几天和同事刚好聊到相关的话题，发现其实大家对这块理解还是比较模糊的。希望这篇文章能够从更深入的角度告诉你为什么，它到底做了什么事？

## 思考

### 一、为什么会中止运行

```
func main() {
	panic("EDDYCJY.")
}
```

输出结果：

```
$ go run main.go
panic: EDDYCJY.

goroutine 1 [running]:
main.main()
	/Users/eddycjy/go/src/github.com/EDDYCJY/awesomeProject/main.go:4 +0x39
exit status 2
```

请思考一下，为什么执行 `panic` 后会导致应用程序运行中止？（而不是单单说执行了 `panic` 所以就结束了这么含糊）

### 二、为什么不会中止运行

```
func main() {
	defer func() {
		if err := recover(); err != nil {
			log.Printf("recover: %v", err)
		}
	}()

	panic("EDDYCJY.")
}
```

输出结果：

```
$ go run main.go 
2019/05/11 23:39:47 recover: EDDYCJY.
```

请思考一下，为什么加上 `defer` + `recover` 组合就可以保护应用程序？

### 三、不设置 defer 行不

上面问题二是 `defer` + `recover` 组合，那我去掉 `defer` 是不是也可以呢？如下：

```
func main() {
	if err := recover(); err != nil {
		log.Printf("recover: %v", err)
	}

	panic("EDDYCJY.")
}
```

输出结果：

```
$ go run main.go
panic: EDDYCJY.

goroutine 1 [running]:
main.main()
	/Users/eddycjy/go/src/github.com/EDDYCJY/awesomeProject/main.go:10 +0xa1
exit status 2
```

竟然不行，啊呀毕竟入门教程都写的 `defer` + `recover` 组合 “万能” 捕获。但是为什么呢。去掉 `defer` 后为什么就无法捕获了？

请思考一下，为什么需要设置 `defer` 后 `recover` 才能起作用？

同时你还需要仔细想想，我们设置 `defer` + `recover` 组合后就能无忧无虑了吗，各种 “乱” 写了吗？

### 四、为什么起个 goroutine 就不行

```
func main() {
	go func() {
		defer func() {
			if err := recover(); err != nil {
				log.Printf("recover: %v", err)
			}
		}()
	}()

	panic("EDDYCJY.")
}
```

输出结果：

```
$ go run main.go 
panic: EDDYCJY.

goroutine 1 [running]:
main.main()
	/Users/eddycjy/go/src/github.com/EDDYCJY/awesomeProject/main.go:14 +0x51
exit status 2
```

请思考一下，为什么新起了一个 `Goroutine` 就无法捕获到异常了？到底发生了什么事...

## 源码

接下来我们将带着上述 4+1 个小思考题，开始对源码的剖析和分析，尝试从阅读源码中找到思考题的答案和更多为什么

### 数据结构

```
type _panic struct {
	argp      unsafe.Pointer
	arg       interface{} 
	link      *_panic 
	recovered bool
	aborted   bool 
}
```

在 `panic` 中是使用 `_panic` 作为其基础单元的，每执行一次 `panic` 语句，都会创建一个 `_panic`。它包含了一些基础的字段用于存储当前的 `panic` 调用情况，涉及的字段如下：

- argp：指向 `defer` 延迟调用的参数的指针
- arg：`panic` 的原因，也就是调用 `panic` 时传入的参数
- link：指向上一个调用的 `_panic`
- recovered：`panic` 是否已经被处理，也就是是否被 `recover`
- aborted：`panic` 是否被中止

另外通过查看 `link` 字段，可得知其是一个链表的数据结构，如下图：

![image](http://wx3.sinaimg.cn/large/006fVPCvly1g2muc73jp1j30hc099q2x.jpg)

### 恐慌 panic

```
func main() {
	panic("EDDYCJY.")
}
```

输出结果：

```
$ go run main.go
panic: EDDYCJY.

goroutine 1 [running]:
main.main()
	/Users/eddycjy/go/src/github.com/EDDYCJY/awesomeProject/main.go:4 +0x39
exit status 2
```

我们去反查一下 `panic` 处理具体逻辑的地方在哪，如下：

```
$ go tool compile -S main.go
"".main STEXT size=66 args=0x0 locals=0x18
	0x0000 00000 (main.go:23)	TEXT	"".main(SB), ABIInternal, $24-0
	0x0000 00000 (main.go:23)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:23)	CMPQ	SP, 16(CX)
	...
	0x002f 00047 (main.go:24)	PCDATA	$2, $0
	0x002f 00047 (main.go:24)	MOVQ	AX, 8(SP)
	0x0034 00052 (main.go:24)	CALL	runtime.gopanic(SB)
```

显然汇编代码直指内部实现是 `runtime.gopanic`，我们一起来看看这个方法做了什么事，如下（省略了部分）：

```
func gopanic(e interface{}) {
	gp := getg()
	...
	var p _panic
	p.arg = e
	p.link = gp._panic
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))
    
	for {
		d := gp._defer
		if d == nil {
			break
		}

		// defer...
		...
		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		p.argp = unsafe.Pointer(getargp(0))
		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
		p.argp = nil

		// recover...
		if p.recovered {
			...
			mcall(recovery)
			throw("recovery failed") // mcall should not return
		}
	}

	preprintpanics(gp._panic)

	fatalpanic(gp._panic) // should not return
	*(*int)(nil) = 0      // not reached
}
```

- 获取指向当前 `Goroutine` 的指针
- 初始化一个 `panic` 的基本单位 `_panic` 用作后续的操作
- 获取当前 `Goroutine` 上挂载的 `_defer`（数据结构也是链表）
- 若当前存在 `defer` 调用，则调用 `reflectcall` 方法去执行先前 `defer` 中延迟执行的代码，若在执行过程中需要运行 `recover` 将会调用 `gorecover` 方法
- 结束前，使用 `preprintpanics` 方法打印出所涉及的 `panic` 消息
- 最后调用 `fatalpanic` 中止应用程序，实际是执行 `exit(2)` 进行最终退出行为的

通过对上述代码的执行分析，可得知 `panic` 方法实际上就是处理当前 `Goroutine(g)` 上所挂载的 `._panic` 链表（所以无法对其他 `Goroutine` 的异常事件响应），然后对其所属的 `defer` 链表和 `recover` 进行检测并处理，最后调用退出命令中止应用程序

### 无法恢复的恐慌 fatalpanic

```
func fatalpanic(msgs *_panic) {
	pc := getcallerpc()
	sp := getcallersp()
	gp := getg()
	var docrash bool

	systemstack(func() {
		if startpanic_m() && msgs != nil {
		    ...
			printpanics(msgs)
		}

		docrash = dopanic_m(gp, pc, sp)
	})

	systemstack(func() {
		exit(2)
	})

	*(*int)(nil) = 0
}
```

我们看到在异常处理的最后会执行该方法，似乎它承担了所有收尾工作。实际呢，它是在最后对程序执行 `exit` 指令来达到中止运行的作用，但在结束前它会通过 `printpanics` 递归输出所有的异常消息及参数。代码如下：

```
func printpanics(p *_panic) {
	if p.link != nil {
		printpanics(p.link)
		print("\t")
	}
	print("panic: ")
	printany(p.arg)
	if p.recovered {
		print(" [recovered]")
	}
	print("\n")
}
```

所以不要以为所有的异常都能够被 `recover` 到，实际上像 `fatal error` 和 `runtime.throw` 都是无法被 `recover` 到的，甚至是 oom 也是直接中止程序的，也有反手就给你来个 `exit(2)` 教做人。因此在写代码时你应该要相对注意些，“恐慌” 是存在无法恢复的场景的

### 恢复 recover

```
func main() {
	defer func() {
		if err := recover(); err != nil {
			log.Printf("recover: %v", err)
		}
	}()

	panic("EDDYCJY.")
}
```

输出结果：

```
$ go run main.go 
2019/05/11 23:39:47 recover: EDDYCJY.
```

和预期一致，成功捕获到了异常。但是 `recover` 是怎么恢复 `panic` 的呢？再看看汇编代码，如下：

```
$ go tool compile -S main.go
"".main STEXT size=110 args=0x0 locals=0x18
	0x0000 00000 (main.go:5)	TEXT	"".main(SB), ABIInternal, $24-0
	...
	0x0024 00036 (main.go:6)	LEAQ	"".main.func1·f(SB), AX
	0x002b 00043 (main.go:6)	PCDATA	$2, $0
	0x002b 00043 (main.go:6)	MOVQ	AX, 8(SP)
	0x0030 00048 (main.go:6)	CALL	runtime.deferproc(SB)
	...
	0x0050 00080 (main.go:12)	CALL	runtime.gopanic(SB)
	0x0055 00085 (main.go:12)	UNDEF
	0x0057 00087 (main.go:6)	XCHGL	AX, AX
	0x0058 00088 (main.go:6)	CALL	runtime.deferreturn(SB)
	...
	0x0022 00034 (main.go:7)	MOVQ	AX, (SP)
	0x0026 00038 (main.go:7)	CALL	runtime.gorecover(SB)
	0x002b 00043 (main.go:7)	PCDATA	$2, $1
	0x002b 00043 (main.go:7)	MOVQ	16(SP), AX
	0x0030 00048 (main.go:7)	MOVQ	8(SP), CX
	...
	0x0056 00086 (main.go:8)	LEAQ	go.string."recover: %v"(SB), AX
	...
	0x0086 00134 (main.go:8)	CALL	log.Printf(SB)
	...
```

通过分析底层调用，可得知主要是如下几个方法：
- runtime.deferproc
- runtime.gopanic
- runtime.deferreturn
- runtime.gorecover

在上小节中，我们讲述了简单的流程，`gopanic` 方法会调用当前 `Goroutine` 下的 `defer` 链表，若 `reflectcall` 执行中遇到 `recover` 就会调用 `gorecover` 进行处理，该方法代码如下：


```
func gorecover(argp uintptr) interface{} {
	gp := getg()
	p := gp._panic
	if p != nil && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}
```

这代码，看上去挺简单的，核心就是修改 `recovered` 字段。该字段是用于标识当前 `panic` 是否已经被 `recover` 处理。但是这和我们想象的并不一样啊，程序是怎么从 `panic` 流转回去的呢？是不是在核心方法里处理了呢？我们再看看 `gopanic` 的代码，如下：


```
func gopanic(e interface{}) {
	...
	for {
		// defer...
		...
		pc := d.pc
		sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
		freedefer(d)
		
		// recover...
		if p.recovered {
			atomic.Xadd(&runningPanicDefers, -1)

			gp._panic = p.link
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}
			if gp._panic == nil { 
				gp.sig = 0
			}

			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			mcall(recovery)
			throw("recovery failed") 
		}
	}
    ...
}
```

我们回到 `gopanic` 方法中再仔细看看，发现实际上是包含对 `recover` 流转的处理代码的。恢复流程如下：

- 判断当前 `_panic` 中的 `recover` 是否已标注为处理
- 从 `_panic` 链表中删除已标注中止的 `panic` 事件，也就是删除已经被恢复的 `panic` 事件
- 将相关需要恢复的栈帧信息传递给 `recovery` 方法的 `gp` 参数（每个栈帧对应着一个未运行完的函数。栈帧中保存了该函数的返回地址和局部变量）
- 执行 `recovery` 进行恢复动作

从流程来看，最核心的是 `recovery` 方法。它承担了异常流转控制的职责。代码如下：

```
func recovery(gp *g) {
	sp := gp.sigcode0
	pc := gp.sigcode1

	if sp != 0 && (sp < gp.stack.lo || gp.stack.hi < sp) {
		print("recover: ", hex(sp), " not in [", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n")
		throw("bad recovery")
	}

	gp.sched.sp = sp
	gp.sched.pc = pc
	gp.sched.lr = 0
	gp.sched.ret = 1
	gogo(&gp.sched)
}
```

粗略一看，似乎就是很简单的设置了一些值？但实际上设置的是编译器中伪寄存器的值，常常被用于维护上下文等。在这里我们需要结合 `gopanic` 方法一同观察 `recovery` 方法。它所使用的栈指针 `sp` 和程序计数器 `pc` 是由当前 `defer` 在调用流程中的 `deferproc` 传递下来的，因此实际上最后是通过 `gogo` 方法跳回了 `deferproc` 方法。另外我们注意到：

```
gp.sched.ret = 1
```

在底层中程序将 `gp.sched.ret` 设置为了 1，也就是**没有实际调用** `deferproc` 方法，直接修改了其返回值。意味着默认它已经处理完成。直接转移到 `deferproc` 方法的下一条指令去。至此为止，异常状态的流转控制就已经结束了。接下来就是继续走 `defer` 的流程了


为了验证这个想法，我们可以看一下核心的跳转方法 `gogo` ，代码如下：

```
// void gogo(Gobuf*)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB),NOSPLIT,$8-4
	MOVW	buf+0(FP), R1
	MOVW	gobuf_g(R1), R0
	BL	setg<>(SB)

	MOVW	gobuf_sp(R1), R13	// restore SP==R13
	MOVW	gobuf_lr(R1), LR
	MOVW	gobuf_ret(R1), R0
	MOVW	gobuf_ctxt(R1), R7
	MOVW	$0, R11
	MOVW	R11, gobuf_sp(R1)	// clear to help garbage collector
	MOVW	R11, gobuf_ret(R1)
	MOVW	R11, gobuf_lr(R1)
	MOVW	R11, gobuf_ctxt(R1)
	MOVW	gobuf_pc(R1), R11
	CMP	R11, R11 // set condition codes for == test, needed by stack split
	B	(R11)
```

通过查看代码可得知其主要作用是从 `Gobuf` 恢复状态。简单来讲就是将寄存器的值修改为对应 `Goroutine(g)` 的值，而在文中讲了很多次的 `Gobuf`，如下：

```
type gobuf struct {
	sp   uintptr
	pc   uintptr
	g    guintptr
	ctxt unsafe.Pointer
	ret  sys.Uintreg
	lr   uintptr
	bp   uintptr
}
```

讲道理，其实它存储的就是 `Goroutine` 切换上下文时所需要的一些东西

## 拓展

```
const(
	OPANIC       // panic(Left)
	ORECOVER     // recover()
	...
)
...
func walkexpr(n *Node, init *Nodes) *Node {
    ...
	switch n.Op {
	default:
		Dump("walk", n)
		Fatalf("walkexpr: switch 1 unknown op %+S", n)

	case ONONAME, OINDREGSP, OEMPTY, OGETG:
	case OTYPE, ONAME, OLITERAL:
	    ...
	case OPANIC:
		n = mkcall("gopanic", nil, init, n.Left)

	case ORECOVER:
		n = mkcall("gorecover", n.Type, init, nod(OADDR, nodfp, nil))
	...
}
```

实际上在调用 `panic` 和 `recover` 关键字时，是在编译阶段先转换为相应的 OPCODE 后，再由编译器转换为对应的运行时方法。并不是你所想像那样一步到位，有兴趣的小伙伴可以研究一下

## 总结

本文主要针对 `panic` 和 `recover` 关键字进行了深入源码的剖析，而开头的 4+1 个思考题，就是希望您能够带着疑问去学习，达到事半功倍的功效

另外本文和 `defer` 有一定的关联性，因此需要有一定的基础知识。若刚刚看的时候这部分不理解，学习后可以再读一遍加深印象

在最后，现在的你可以回答这几个思考题了吗？说出来了才是真的懂 ：）
