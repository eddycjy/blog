---
title: "Go 并发：一些有趣的现象和要避开的 “坑”"
date: 2020-12-10T00:25:59+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

最近在看 Go 并发相关的内容，发现还是有不少细节容易让人迷迷糊糊的，一个不小心就踏入深坑里，且指不定要在上线跑了一些数据后才能发现，那可真是太人崩溃了。

今天来分享几个案例，希望大家在编码时能够避开这几个 “坑”。

## 案例一

### 演示代码

第一个案例来自 @鸟窝 大佬在极客时间的分享，代码如下：

```
func main() {
	count := 0
	wg := sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				count++
			}
		}()
	}
	wg.Wait()

	fmt.Println(count)
}
```

思考一下，最后输出的 `count` 变量的值是多少？是不是一百万？

### 输出结果

在上述代码中，我们通过 `for-loop ` 循环起 `goroutine` 进行自增，并使用了 `sync.WaitGroup` 来保证所有的 goroutine 都执行完毕才输出最终的结果值。

最终的输出结果如下：

```
// 第一次执行
638853

// 第二次执行
654473

// 第三次执行
786193
```

输出的结果值不是恒定的，也就是每次输出的都不一样，且基本不会达到想象中的一百万。

### 分析原因

其原因在于 `count++` 并不是一个原子操作，在汇编上就包含了好几个动作，如下：

```
MOVQ "".count(SB), AX 
LEAQ 1(AX), CX 
MOVQ CX, "".count(SB)
```

因为可能会同时存在多个 goroutine 同时读取到 `count` 的值为 1212，并各自自增 1，再将其写回。

与此同时也会有其他的 goroutine 可能也在其自增时读到了值，形成了互相覆盖的情况，这是一种并发访问共享数据的错误。

### 发现问题

这类竞争问题可以通过 Go 语言所提供的的 race 检测（[Go race detector](https://blog.golang.org/race-detector)）来进行分析和发现：

```
$ go run -race main.go 
==================
WARNING: DATA RACE
Read at 0x00c0000c6008 by goroutine 13:
  main.main.func1()
      /Users/eddycjy/go-application/awesomeProject/main.go:28 +0x78

Previous write at 0x00c0000c6008 by goroutine 7:
  main.main.func1()
      /Users/eddycjy/go-application/awesomeProject/main.go:28 +0x91

Goroutine 13 (running) created at:
  main.main()
      /Users/eddycjy/go-application/awesomeProject/main.go:25 +0xe4

Goroutine 7 (running) created at:
  main.main()
      /Users/eddycjy/go-application/awesomeProject/main.go:25 +0xe4
==================
...
489194
Found 3 data race(s)
exit status 66
```

编译器会通过探测所有的内存访问，监听其内存地址的访问（读或写）。在应用运行时就能够发现对共享变量的访问和操作，进而发现问题并打印出相关的警告信息。

需要注意的一点是，`go run -race` 是运行时检测，并不是编译时。且 race 存在明确的性能开销，通常是正常程序的十倍，因此不要想不开在生产环境打开这个配置，很容易翻车。

## 案例二

### 演示代码

第二个案例来自煎鱼在脑子的分享，代码如下：

```
func main() {
	wg := sync.WaitGroup{}
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go func(i int) {
			defer wg.Done()
			fmt.Println(i)
		}(i)
	}
	wg.Wait()
}
```

思考一下，最后输出的结果是什么？值都是 4 吗？输出是稳定有序的吗？

### 输出结果

在上述代码中，我们通过 `for-loop` 循环起了多个 `goroutine`，并将变量 `i` 作为形参传递给了 `goroutine`，最后在 `goroutine` 内输出了变量 `i`。

最终的输出结果如下：

```
// 第一次输出
0
1
2
4
3

// 第二次输出
4
0
1
2
3
```

显然，从结果上来看，输出的值都是无序且不稳定的，值更不是 4。这到底是为什么？

### 分析原因

其原因在于，即使所有的 `goroutine` 都创建完了，但 `goroutine` 不一定已经开始运行了。

也就是等到 `goroutine` 真正去执行输出时，变量 `i` （值拷贝）可能已经不是创建时的值了。

其整个程序扭转实质上分为了多个阶段，也就是各自运行的时间线并不同，可以其拆分为：

- 先创建：`for-loop` 循环创建 `goroutine`。

- 再调度：协程`goroutine` 开始调度执行。

- 才执行：开始执行 `goroutine` 内的输出。

同时 `goroutine` 的调度存在一定的随机性（建议了解一下 GMP 模型），那么其输出的结果就势必是无序且不稳定的。

### 发现问题

这时候你可能会想，那前面提到的 `go run -race` 能不能发现这个问题呢。如下：

```
$ go run -race main.go
0
1
2
3
4
```

没有出现警告，显然是不能的，因为其本质上并不是并发访问共享数据的错误，且会导致程序变成了串行，从而蒙蔽了你的双眼。

## 案例三

### 演示代码

第三个案例来自煎鱼在梦里的分享，代码如下：

```
func main() {
	wg := sync.WaitGroup{}
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go func() {
			defer wg.Done()
			fmt.Println(i)
		}()
	}
	wg.Wait()
}
```

思考一下，最后输出的结果是什么？值都是 4 吗？会像案例二一样乱窜吗？

### 输出结果

在上述代码中，与案例二大体没有区别，主要是变量 `i` 没有作为形参传入。

最终的输出结果如下：

```
// 第一次输出
5
5
5
5
5
```

初步从输出的结果上来看都是 5，这时候就会有人迷糊了，为什么不是 4 呢？

不少人会因不是 4 而陷入了迷惑，但千万不要被一两次的输出迷惑了心智，认为铁定就是 5 了。可以再动手多输出几次，如下：

```
// 多输出几次
5
3
5
5
5
```

最终会发现...输出结果存在随机性，输出结果并不是 100% 都是 5，更不用提 4 了。这到底是为什么呢？

### 分析原因

其原因与案例二其实非常接近，理论上理解了案例二也就能解决案例三。

其本质还是创建 `goroutine` 与真正执行 `fmt.Println` 并不同步。因此很有可能在你执行 `fmt.Println` 时，循环 `for-loop` 已经运行完毕，因此变量 `i` 的值最终变成了 5。

那么相反，其也有可能没运行完，存在随机性。写个 test case 就能发现明显的不同。

## 总结

在本文中，我分享了几个近期看到次数最频繁的一些并发上的小 “坑”，希望对你有所帮助。同时你也可以回想一下，在你编写 Go 并发程序有没有也遇到过什么问题？

同时你也可以回想一下，在你编写 Go 并发程序有没有也遇到过什么问题？

欢迎大家一起讨论交流。