---

title:      "来，控制一下 goroutine 的并发数量"
date:       2019-01-20 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
---

![image](https://s2.ax1x.com/2020/02/27/3wnOsJ.jpg)

## 问题

```go
func main() {
	userCount := math.MaxInt64
	for i := 0; i < userCount; i++ {
		go func(i int) {
		    // 做一些各种各样的业务逻辑处理
			fmt.Printf("go func: %d\n", i)
			time.Sleep(time.Second)
		}(i)
	}
}
```

在这里，假设 `userCount` 是一个外部传入的参数（不可预测，有可能值非常大），有人会全部丢进去循环。想着全部都并发 goroutine 去同时做某一件事。觉得这样子会效率会更高，对不对！

那么，你觉得这里有没有什么问题？

## 噩梦般的开始

当然，在**特定场景下**，问题可大了。因为在本文被丢进去同时并发的可是一个极端值。我们可以一起观察下图的指标分析，看看情况有多 “崩溃”。下图是上述代码的表现：

### 输出结果

```
...
go func: 5839
go func: 5840
go func: 5841
go func: 5842
go func: 5915
go func: 5524
go func: 5916
go func: 8209
go func: 8264
signal: killed
```

如果你自己执行过代码，在 “输出结果” 上你会遇到如下问题：

- 系统资源占用率不断上涨
- 输出一定数量后：控制台就不再刷新输出最新的值了
- 信号量：signal: killed

### 系统负载

![image](https://s2.ax1x.com/2020/02/27/3wnxd1.jpg)

### CPU

![image](https://s2.ax1x.com/2020/02/27/3wuKW8.jpg)

短时间内系统负载暴增

### 虚拟内存

![image](https://s2.ax1x.com/2020/02/27/3wu1yQ.jpg)

短时间内占用的虚拟内存暴增

### top

```
PID    COMMAND      %CPU  TIME     #TH   #WQ  #PORT MEM    PURG   CMPRS  PGRP  PPID  STATE    BOOSTS
...
73414  test         100.2 01:59.50 9/1   0    18    6801M+ 0B     114G+  73403 73403 running  *0[1]
```

### 小结

如果仔细看过监控工具的示意图，就可以知道其实我间隔的执行了两次，能看到系统间的使用率幅度非常大。当进程被杀掉后，整体又恢复为正常值

在这里，我们回到主题，就是在**不控制并发的 goroutine 数量** 会发生什么问题？大致如下：

- CPU 使用率浮动上涨
- Memory 占用不断上涨。也可以看看 CMPRS，它表示进程的压缩数据的字节数。已经到达 114G+ 了
- 主进程崩溃（被杀掉了）

简单来说，“崩溃” 的原因就是对系统资源的占用过大。常见的比如：打开文件数（too many files open）、内存占用等等

### 危害

对该台服务器产生非常大的影响，影响自身及相关联的应用。很有可能导致不可用或响应缓慢，另外启动了复数 “失控” 的 goroutine，导致程序流转混乱

## 解决方案

在前面花了大量篇幅，渲染了在存在大量并发 goroutine 数量时，不控制的话会出现 “严重” 的问题，接下来一起思考下解决方案。如下：

1. 控制/限制 goroutine 同时并发运行的数量
2. 改变应用程序的逻辑写法（避免大规模的使用系统资源和等待）
3. ~~调整服务的硬件配置、最大打开数、内存等阈值~~

## 控制 goroutine 并发数量

接下来正式的开始解决这个问题，希望你认真阅读的同时加以思考，因为这个问题在实际项目中真的是太常见了！

问题已经抛出来了，你需要做的是**想想有什么办法**解决这个问题。建议你自行思考一下技术方案。再接着往下看 :-)

### 尝试 chan

```go
func main() {
	userCount := 10
	ch := make(chan bool, 2)
	for i := 0; i < userCount; i++ {
		ch <- true
		go Read(ch, i)
	}

	//time.Sleep(time.Second)
}

func Read(ch chan bool, i int) {
	fmt.Printf("go func: %d\n", i)
	<- ch
}
```

输出结果：

```
go func: 1
go func: 2
go func: 3
go func: 4
go func: 5
go func: 6
go func: 7
go func: 8
go func: 0
```

嗯，我们似乎很好的控制了 2 个 2 个的 “顺序” 执行多个 goroutine。但是，问题出现了。你仔细数一下输出结果，才 9 个值？

这明显就不对。原因出在当主协程结束时，子协程也是会被终止掉的。因此剩余的 goroutine 没来及把值输出，就被送上路了（不信你把 `time.Sleep` 打开看看，看看输出数量）

### 尝试 sync

```go
...
var wg = sync.WaitGroup{}

func main() {
	userCount := 10
	for i := 0; i < userCount; i++ {
		wg.Add(1)
		go Read(i)
	}

	wg.Wait()
}

func Read(i int) {
	defer wg.Done()
	fmt.Printf("go func: %d\n", i)
}
```

嗯，单纯的使用 `sync.WaitGroup` 也不行。没有控制到同时并发的 goroutine 数量（代指达不到本文所要求的目标）

#### 小结

单纯**简单**使用 channel 或 sync 都有明显缺陷，不行。我们再看看组件配合能不能实现

### 尝试 chan + sync

```go
...
var wg = sync.WaitGroup{}

func main() {
	userCount := 10
	ch := make(chan bool, 2)
	for i := 0; i < userCount; i++ {
		wg.Add(1)
		go Read(ch, i)
	}

	wg.Wait()
}

func Read(ch chan bool, i int) {
	defer wg.Done()

	ch <- true
	fmt.Printf("go func: %d, time: %d\n", i, time.Now().Unix())
	time.Sleep(time.Second)
	<-ch
}
```

输出结果：

```
go func: 9, time: 1547911938
go func: 1, time: 1547911938
go func: 6, time: 1547911939
go func: 7, time: 1547911939
go func: 8, time: 1547911940
go func: 0, time: 1547911940
go func: 3, time: 1547911941
go func: 2, time: 1547911941
go func: 4, time: 1547911942
go func: 5, time: 1547911942
```

从输出结果来看，确实实现了控制 goroutine 以 2 个 2 个的数量去执行我们的 “业务逻辑”，当然结果集也理所应当的是乱序输出

### 方案一：简单 Semaphore

在确立了简单使用 chan + sync 的方案是可行后，我们重新将流转逻辑封装为 [gsema](https://github.com/EDDYCJY/gsema)，主程序变成如下：

```go
import (
	"fmt"
	"time"

	"github.com/EDDYCJY/gsema"
)

var sema = gsema.NewSemaphore(3)

func main() {
	userCount := 10
	for i := 0; i < userCount; i++ {
		go Read(i)
	}

	sema.Wait()
}

func Read(i int) {
	defer sema.Done()
	sema.Add(1)

	fmt.Printf("go func: %d, time: %d\n", i, time.Now().Unix())
	time.Sleep(time.Second)
}
```

### 分析方案

在上述代码中，程序执行流程如下：

- 设置允许的并发数目为 3 个
- 循环 10 次，每次启动一个 goroutine 来执行任务
- 每一个 goroutine 在内部利用 `sema` 进行调控是否阻塞
- 按允许并发数逐渐释出 goroutine，最后结束任务

看上去人模人样，没什么严重问题。但却有一个 “大” 坑，认真看到第二点 “每次启动一个 goroutine” 这句话。这里**有点问题**，提前产生那么多的 goroutine 会不会有什么问题，接下来一起分析下利弊，如下：

#### 利

- 适合**量不大、复杂度低**的使用场景
  - 几百几千个、几十万个也是可以接受的（看具体业务场景）
  - 实际业务逻辑在运行前就已经被阻塞等待了（因为并发数受限），基本实际业务逻辑损耗的性能比 goroutine 本身大
  - goroutine 本身很轻便，仅损耗极少许的内存空间和调度。这种等待响应的情况都是躺好了，等待任务唤醒
- Semaphore 操作复杂度低且流转简单，容易控制

#### 弊

- 不适合**量很大、复杂度高**的使用场景
  - 有几百万、几千万个 goroutine 的话，就浪费了大量调度 goroutine 和内存空间。恰好你的服务器也接受不了的话
- Semaphore 操作复杂度提高，要管理更多的状态

### 小结

- 基于什么业务场景，就用什么方案去做事
- 有足够的时间，允许你去追求更优秀、极致的方案（用第三方库也行）

用哪种方案，我认为主要基于以上两点去思考，都是 OK 的。没有对错，只有当前业务场景能不能接受，这个预先启动的 goroutine 数量你的系统是否能够接受

当然了，常见/简单的 Go 应用采用这类技术方案，基本就能解决问题了。因为像本文第一节 “问题” 如此超巨大数量的情况，情况很少。其并不存在那些 “特殊性”。因此用这个方案基本 OK

## 灵活控制 goroutine 并发数量

小手一紧。隔壁老王发现了新的问题。“方案一” 中，在**输入输出一体**的情况下，在常见的业务场景中确实可以

但，这次新的业务场景比较特殊，要控制输入的数量，以此达到**改变允许并发运行 goroutine 的数量**。我们仔细想想，要做出如下改变：

- 输入/输出要抽离，才可以分别控制
- 输入/输出要可变，理所应当在 for-loop 中（可设置数值的地方）
- 允许改变 goroutine 并发数量，但它也必须有一个**最大值**（因为允许改变是相对）

### 方案二：灵活 chan + sync

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup

func main() {
	userCount := 10
	ch := make(chan int, 5)
	for i := 0; i < userCount; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for d := range ch {
				fmt.Printf("go func: %d, time: %d\n", d, time.Now().Unix())
				time.Sleep(time.Second * time.Duration(d))
			}
		}()
	}

	for i := 0; i < 10; i++ {
		ch <- 1
		ch <- 2
		//time.Sleep(time.Second)
	}

	close(ch)
	wg.Wait()
}
```

输出结果：

```
...
go func: 1, time: 1547950567
go func: 3, time: 1547950567
go func: 1, time: 1547950567
go func: 2, time: 1547950567
go func: 2, time: 1547950567
go func: 3, time: 1547950567
go func: 1, time: 1547950568
go func: 2, time: 1547950568
go func: 3, time: 1547950568
go func: 1, time: 1547950568
go func: 3, time: 1547950569
go func: 2, time: 1547950569
```

在 “方案二” 中，我们可以随时随地的根据新的业务需求，做如下事情：

- 变更 channel 的输入数量
- 能够根据特殊情况，变更 channel 的循环值
- 变更最大允许并发的 goroutine 数量

总的来说，就是可控空间都尽量放开了，是不是更加灵活了呢 :-)

### 方案三：第三方库

- [go-playground/pool](https://github.com/go-playground/pool)
- [nozzle/throttler](https://github.com/nozzle/throttler)
- [Jeffail/tunny](https://github.com/Jeffail/tunny)
- [panjf2000/ants](https://github.com/panjf2000/ants)

比较成熟的第三方库也不少，基本都是以生成和管理 goroutine 为目标的池工具。我简单列了几个，具体建议大家阅读下源码或者多找找，原理相似

## 总结

在本文的开头，我花了大力气（极端数量），告诉你**同时并发过多的 goroutine 数量会导致系统占用资源不断上涨。最终该服务崩盘的极端情况**。为的是希望你今后避免这种问题，给你留下深刻的印象

接下来我们以 “控制 goroutine 并发数量” 为主题，展开了一番分析。分别给出了三种方案。在我看来，各具优缺点，我建议你**挑选合适自身场景的技术方案**就可以了

因为，有不同类型的技术方案也能解决这个问题，千人千面。本文推荐的是较常见的解决方案，也欢迎大家在评论区继续补充 :-)
