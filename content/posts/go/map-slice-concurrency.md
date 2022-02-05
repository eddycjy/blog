---
title: "为什么 Go map 和 slice 是非线程安全的？"
date: 2021-12-31T12:54:49+08:00
toc: true
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

初入 Go 语言的大门，有不少的小伙伴会快速的 3 天精通 Go，5 天上手项目，14 天上线业务迭代，21 天排查、定位问题，顺带捎个反省报告。

其中最常见的初级错误，Go 面试较最爱问的问题之一：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d1a0506617a4729abe35927d87508d0~tplv-k3u1fbpfcp-zoom-1.image)
（来自读者提问）

为什么在 Go 语言里，map 和 slice 不支持并发读写，也就是是非线程安全的，为什么不支持？

见招拆招后，紧接着就会开始讨论如何让他们俩 ”冤家“ 支持并发读写？

今天我们这篇文章就来理一理，了解其前因后果，一起吸鱼学懂 Go 语言。

非线程安全的例子
--------

### slice

我们使用多个 goroutine 对类型为 slice 的变量进行操作，看看结果会变的怎么样。

如下：

```
func main() {
 var s []string
 for i := 0; i < 9999; i++ {
  go func() {
   s = append(s, "脑子进煎鱼了")
  }()
 }

 fmt.Printf("进了 %d 只煎鱼", len(s))
}

```

输出结果：

```
// 第一次执行
进了 5790 只煎鱼
// 第二次执行
进了 7370 只煎鱼
// 第三次执行
进了 6792 只煎鱼

```

你会发现无论你执行多少次，每次输出的值大概率都不会一样。也就是追加进 slice 的值，出现了覆盖的情况。

因此在循环中所追加的数量，与最终的值并不相等。且这种情况，是不会报错的，是一个出现率不算高的隐式问题。

这个产生的主要原因是程序逻辑本身就有问题，同时读取到相同索引位，自然也就会产生覆盖的写入了。

### map

同样针对 map 也如法炮制一下。重复针对类型为 map 的变量进行写入。

如下：

```
func main() {
 s := make(map[string]string)
 for i := 0; i < 99; i++ {
  go func() {
   s["煎鱼"] = "吸鱼"
  }()
 }

 fmt.Printf("进了 %d 只煎鱼", len(s))
}

```

输出结果：

```
fatal error: concurrent map writes

goroutine 18 [running]:
runtime.throw(0x10cb861, 0x15)
        /usr/local/Cellar/go/1.16.2/libexec/src/runtime/panic.go:1117 +0x72 fp=0xc00002e738 sp=0xc00002e708 pc=0x1032472
runtime.mapassign_faststr(0x10b3360, 0xc0000a2180, 0x10c91da, 0x6, 0x0)
        /usr/local/Cellar/go/1.16.2/libexec/src/runtime/map_faststr.go:211 +0x3f1 fp=0xc00002e7a0 sp=0xc00002e738 pc=0x1011a71
main.main.func1(0xc0000a2180)
        /Users/eddycjy/go-application/awesomeProject/main.go:9 +0x4c fp=0xc00002e7d8 sp=0xc00002e7a0 pc=0x10a474c
runtime.goexit()
        /usr/local/Cellar/go/1.16.2/libexec/src/runtime/asm_amd64.s:1371 +0x1 fp=0xc00002e7e0 sp=0xc00002e7d8 pc=0x1063fe1
created by main.main
        /Users/eddycjy/go-application/awesomeProject/main.go:8 +0x55

```

好家伙，程序运行会直接报错。并且是 Go 源码调用 `throw` 方法所导致的致命错误，也就是说 Go 进程会中断。

不得不说，这个并发写 map 导致的 `fatal error: concurrent map writes` 错误提示。我有一个朋友，已经看过少说几十次了，不同组，不同人...

是个日经的隐式问题。

如何支持并发读写
--------

### 对 map 上锁

实际上我们仍然存在并发读写 map 的诉求（程序逻辑决定），因为 Go 语言中的 goroutine 实在是太方便了。

像是一般写爬虫任务时，基本会用到多个 goroutine，获取到数据后再写入到 map 或者 slice 中去。

Go 官方在 Go maps in action 中提供了一种简单又便利的方式来实现：

```
var counter = struct{
    sync.RWMutex
    m map[string]int
}{m: make(map[string]int)}

```

这条语句声明了一个变量，它是一个匿名结构（struct）体，包含一个原生和一个嵌入读写锁 `sync.RWMutex`。

要想从变量中中读出数据，则调用读锁：

```
counter.RLock()
n := counter.m["煎鱼"]
counter.RUnlock()
fmt.Println("煎鱼:", n)

```

要往变量中写数据，则调用写锁：

```
counter.Lock()
counter.m["煎鱼"]++
counter.Unlock()

```

这就是一个最常见的 Map 支持并发读写的方式了。

### sync.Map

#### 前言

虽然有了 Map+Mutex 的极简方案，但是也仍然存在一定问题。那就是在 map 的数据量非常大时，只有一把锁（Mutex）就非常可怕了，一把锁会导致大量的争夺锁，导致各种冲突和性能低下。

常见的解决方案是分片化，将一个大 map 分成多个区间，各区间使用多个锁，这样子锁的粒度就大大降低了。不过该方案实现起来很复杂，很容易出错。因此 Go 团队到比较为止暂无推荐，而是采取了其他方案。

该方案就是在 Go1.9 起支持的 `sync.Map`，其支持并发读写 map，起到一个补充的作用。

#### 具体介绍

Go 语言的 `sync.Map` 支持并发读写 map，采取了 “空间换时间” 的机制，冗余了两个数据结构，分别是：read 和 dirty，减少加锁对性能的影响：

```
type Map struct {
 mu Mutex
 read atomic.Value // readOnly
 dirty map[interface{}]*entry
 misses int
}

```

其是专门为 `append-only` 场景设计的，也就是适合读多写少的场景。这是他的优点之一。

若出现写多/并发多的场景，会导致 read map 缓存失效，需要加锁，冲突变多，性能急剧下降。这是他的重大缺点。

提供了以下常用方法：

```
func (m *Map) Delete(key interface{})
func (m *Map) Load(key interface{}) (value interface{}, ok bool)
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool)
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)
func (m *Map) Range(f func(key, value interface{}) bool)
func (m *Map) Store(key, value interface{})

```

*   Delete：删除某一个键的值。
    
*   Load：返回存储在 map 中的键的值，如果没有值，则返回 nil。ok 结果表示是否在 map 中找到了值。
    
*   LoadAndDelete：删除一个键的值，如果有的话返回之前的值。
    
*   LoadOrStore：如果存在的话，则返回键的现有值。否则，它存储并返回给定的值。如果值被加载，加载的结果为 true，如果被存储，则为 false。
    
*   Range：递归调用，对 map 中存在的每个键和值依次调用闭包函数 `f`。如果 `f` 返回 false 就停止迭代。
    
*   Store：存储并设置一个键的值。
    

实际运行例子如下：

```
var m sync.Map

func main() {
 //写入
 data := []string{"煎鱼", "咸鱼", "烤鱼", "蒸鱼"}
 for i := 0; i < 4; i++ {
  go func(i int) {
   m.Store(i, data[i])
  }(i)
 }
 time.Sleep(time.Second)

 //读取
 v, ok := m.Load(0)
 fmt.Printf("Load: %v, %v\n", v, ok)

 //删除
 m.Delete(1)

 //读或写
 v, ok = m.LoadOrStore(1, "吸鱼")
 fmt.Printf("LoadOrStore: %v, %v\n", v, ok)

 //遍历
 m.Range(func(key, value interface{}) bool {
  fmt.Printf("Range: %v, %v\n", key, value)
  return true
 })
}

```

输出结果：

```
Load: 煎鱼, true
LoadOrStore: 吸鱼, false
Range: 0, 煎鱼
Range: 1, 吸鱼
Range: 3, 蒸鱼
Range: 2, 烤鱼

```

为什么不支持
------

Go Slice 的话，主要还是索引位覆写问题，这个就不需要纠结了，势必是程序逻辑在编写上有明显缺陷，自行改之就好。

但 Go map 就不大一样了，很多人以为是默认支持的，一个不小心就翻车，这么的常见。那凭什么 Go 官方还不支持，难不成太复杂了，性能太差了，到底是为什么？

原因如下（via @go faq）：

*   典型使用场景：map 的典型使用场景是不需要从多个 goroutine 中进行安全访问。
    
*   非典型场景（需要原子操作）：map 可能是一些更大的数据结构或已经同步的计算的一部分。
    
*   性能场景考虑：若是只是为少数程序增加安全性，导致 map 所有的操作都要处理 mutex，将会降低大多数程序的性能。
    

汇总来讲，就是 Go 官方在经过了长时间的讨论后，认为 Go map 更应适配典型使用场景，而不是为了小部分情况，导致大部分程序付出代价（性能），决定了不支持。


## 鼓励

若有任何疑问欢迎评论区反馈和交流，**最好的关系是互相成就**，各位的**点赞**就是[煎鱼](https://github.com/eddycjy)创作的最大动力，感谢支持。

> 文章持续更新，可以微信搜【脑子进煎鱼了】阅读，本文 **GitHub** [github.com/eddycjy/blog](https://github.com/eddycjy/blog) 已收录，欢迎 Star 催更。


总结
--

在今天这篇文章中，我们针对 Go 语言中的 map 和 slice 进行了基本的介绍，也对不支持并发读者的场景进行了模拟展示。

同时也针对业内常见的支持并发读写的方式进行了讲述，最后分析了不支持的原因，让我们对整个前因后果有了一个完整的了解。

不知道你**在日常是否有遇到过 Go 语言中非线性安全的问题呢，欢迎你在评论区留言和大家一起交流**！

