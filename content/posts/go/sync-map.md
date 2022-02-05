---
title: "Go 并发读写 sync.map 的强大之处"
date: 2021-12-31T12:54:50+08:00
toc: true
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

在之前的 《[为什么 Go map 和 slice 是非线程安全的？](http://mp.weixin.qq.com/s?__biz=MzUxMDI4MDc1NA==&mid=2247489045&idx=1&sn=197bda427246e16907c7b471a5dc0572&chksm=f9040348ce738a5ebf541954a4de29ce746238ab7f6e5a2af8a1765c5383ad4208f43b2bac4f&scene=21#wechat_redirect)》 文章中，我们讨论了 Go 语言的 map 和 slice 非线程安全的问题，基于此引申出了 map 的两种目前在业界使用的最多的并发支持的模式。

分别是：

*   原生 map + 互斥锁或读写锁 mutex。
    
*   标准库 sync.Map（Go1.9及以后）。
    

有了选择，总是有选择困难症的，这**两种到底怎么选，谁的性能更加的好**？我有一个朋友说 标准库 sync.Map 性能菜的很，不要用。我到底听谁的...

今天煎鱼就带你揭秘 Go sync.map，我们先会了解清楚什么场景下，Go map 的多种类型怎么用，谁的性能最好！

接着根据各 map 性能分析的结果，针对性的对 sync.map 进行源码解剖，了解 WHY。

一起愉快地开始吸鱼之路。

sync.Map 优势
-----------

在 Go 官方文档中明确指出 Map 类型的一些建议：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6500ee41a08a415ca681fa427f5032b3~tplv-k3u1fbpfcp-zoom-1.image)

*   多个 goroutine 的并发使用是安全的，不需要额外的锁定或协调控制。
    
*   大多数代码应该使用原生的 map，而不是单独的锁定或协调控制，以获得更好的类型安全性和维护性。
    

同时 Map 类型，还针对以下场景进行了性能优化：

*   当一个给定的键的条目只被写入一次但被多次读取时。例如在仅会增长的缓存中，就会有这种业务场景。
    
*   当多个 goroutines 读取、写入和覆盖不相干的键集合的条目时。
    

这两种情况与 Go map 搭配单独的 Mutex 或 RWMutex 相比较，使用 Map 类型可以大大减少锁的争夺。

性能测试
----

听官方文档介绍了一堆好处后，他并没有讲到缺点，所说的性能优化后的优势又是否真实可信。我们一起来验证一下。

首先我们定义基本的数据结构：

```
// 代表互斥锁
type FooMap struct {
 sync.Mutex
 data map[int]int
}

// 代表读写锁
type BarRwMap struct {
 sync.RWMutex
 data map[int]int
}

var fooMap *FooMap
var barRwMap *BarRwMap
var syncMap *sync.Map

// 初始化基本数据结构
func init() {
 fooMap = &FooMap{data: make(map[int]int, 100)}
 barRwMap = &BarRwMap{data: make(map[int]int, 100)}
 syncMap = &sync.Map{}
}

```

在配套方法上，常见的增删改查动作我们都编写了相应的方法。用于后续的压测（只展示部分代码）：

```
func builtinRwMapStore(k, v int) {
 barRwMap.Lock()
 defer barRwMap.Unlock()
 barRwMap.data[k] = v
}

func builtinRwMapLookup(k int) int {
 barRwMap.RLock()
 defer barRwMap.RUnlock()
 if v, ok := barRwMap.data[k]; !ok {
  return -1
 } else {
  return v
 }
}

func builtinRwMapDelete(k int) {
 barRwMap.Lock()
 defer barRwMap.Unlock()
 if _, ok := barRwMap.data[k]; !ok {
  return
 } else {
  delete(barRwMap.data, k)
 }
}

```

其余的类型方法基本类似，考虑重复篇幅问题因此就不在此展示了。

压测方法基本代码如下：

```
func BenchmarkBuiltinRwMapDeleteParalell(b *testing.B) {
 b.RunParallel(func(pb *testing.PB) {
  r := rand.New(rand.NewSource(time.Now().Unix()))
  for pb.Next() {
   k := r.Intn(100000000)
   builtinRwMapDelete(k)
  }
 })
}

```

这块主要就是增删改查的代码和压测方法的准备，压测代码直接复用的是大白大佬的 go19-examples/benchmark-for-map 项目。

也可以使用 Go 官方提供的 map\_bench\_test.go，有兴趣的小伙伴可以自己拉下来运行试一下。

### 压测结果

1）写入：

| 方法名 | 含义 | 压测结果 |
| --- | --- | --- |
| BenchmarkBuiltinMapStoreParalell-4 | map+mutex 写入元素 | 237.1 ns/op |
| BenchmarkSyncMapStoreParalell-4 | sync.map 写入元素 | 509.3 ns/op |
| BenchmarkBuiltinRwMapStoreParalell-4 | map+rwmutex 写入元素 | 207.8 ns/op |

在写入元素上，最慢的是 `sync.map` 类型，其次是原生 map+互斥锁（Mutex），最快的是原生 map+读写锁（RwMutex）。

总体的排序（从慢到快）为：SyncMapStore < MapStore < RwMapStore。

2）查找：

| 方法名 | 含义 | 压测结果 |
| --- | --- | --- |
| BenchmarkBuiltinMapLookupParalell-4 | map+mutex 查找元素 | 166.7 ns/op |
| BenchmarkBuiltinRwMapLookupParalell-4 | map+rwmutex 查找元素 | 60.49 ns/op |
| BenchmarkSyncMapLookupParalell-4 | sync.map 查找元素 | 53.39 ns/op |

在查找元素上，最慢的是原生 map+互斥锁，其次是原生 map+读写锁。最快的是 `sync.map` 类型。

总体的排序为：MapLookup < RwMapLookup < SyncMapLookup。

3）删除：

| 方法名 | 含义 | 压测结果 |
| --- | --- | --- |
| BenchmarkBuiltinMapDeleteParalell-4 | map+mutex 删除元素 | 168.3 ns/op |
| BenchmarkBuiltinRwMapDeleteParalell-4 | map+rwmutex 删除元素 | 188.5 ns/op |
| BenchmarkSyncMapDeleteParalell-4 | sync.map 删除元素 | 41.54 ns/op |

在删除元素上，最慢的是原生 map+读写锁，其次是原生 map+互斥锁，最快的是 `sync.map` 类型。

总体的排序为：RwMapDelete < MapDelete < SyncMapDelete。

### 场景分析

根据上述的压测结果，我们可以得出 `sync.Map` 类型：

*   在读和删场景上的性能是最佳的，领先一倍有多。
    
*   在写入场景上的性能非常差，落后原生 map+锁整整有一倍之多。
    

因此在实际的业务场景中。假设是读多写少的场景，会更建议使用 `sync.Map` 类型。

但若是那种写多的场景，例如多 goroutine 批量的循环写入，那就建议另辟途径了，性能不忍直视（无性能要求另当别论）。

sync.Map 剖析
-----------

清楚如何测试，测试的结果后。我们需要进一步深挖，知其所以然。

为什么 `sync.Map` 类型的测试结果这么的 “偏科”，为什么读操作性能这么高，写操作性能低的可怕，他是怎么设计的？

### 数据结构

`sync.Map` 类型的底层数据结构如下：

```
type Map struct {
 mu Mutex
 read atomic.Value // readOnly
 dirty map[interface{}]*entry
 misses int
}

// Map.read 属性实际存储的是 readOnly。
type readOnly struct {
 m       map[interface{}]*entry
 amended bool
}

```

*   mu：互斥锁，用于保护 read 和 dirty。
    
*   read：只读数据，支持并发读取（atomic.Value 类型）。如果涉及到更新操作，则只需要加锁来保证数据安全。
    

*   read 实际存储的是 readOnly 结构体，内部也是一个原生 map，amended 属性用于标记 read 和 dirty 的数据是否一致。
    

*   dirty：读写数据，是一个原生 map，也就是非线程安全。操作 dirty 需要加锁来保证数据安全。
    
*   misses：统计有多少次读取 read 没有命中。每次 read 中读取失败后，misses 的计数值都会加 1。
    

在 read 和 dirty 中，都有涉及到的结构体：

```
type entry struct {
 p unsafe.Pointer // *interface{}
}

```

其包含一个指针 p, 用于指向用户存储的元素（key）所指向的 value 值。

在此建议你必须搞懂 read、dirty、entry，再往下看，食用效果会更佳，后续会围绕着这几个概念流转。

### 查找过程

划重点，Map 类型本质上是有两个 “map”。一个叫 read、一个叫 dirty，长的也差不多：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a34f8cbe34f428c93bddfa7d302a2bb~tplv-k3u1fbpfcp-zoom-1.image)

sync.Map 的 2 个 map

当我们从 sync.Map 类型中读取数据时，其会先查看 read 中是否包含所需的元素：

*   若有，则通过 atomic 原子操作读取数据并返回。
    
*   若无，则会判断 `read.readOnly` 中的 amended 属性，他会告诉程序 dirty 是否包含 `read.readOnly.m` 中没有的数据；因此若存在，也就是 amended 为 true，将会进一步到 dirty 中查找数据。
    

sync.Map 的读操作性能如此之高的原因，就在于存在 read 这一巧妙的设计，其作为一个缓存层，提供了快路径（fast path）的查找。

同时其结合 amended 属性，配套解决了每次读取都涉及锁的问题，实现了读这一个使用场景的高性能。

### 写入过程

我们直接关注 `sync.Map` 类型的 Store 方法，该方法的作用是新增或更新一个元素。

源码如下：

```
func (m *Map) Store(key, value interface{}) {
 read, _ := m.read.Load().(readOnly)
 if e, ok := read.m[key]; ok && e.tryStore(&value) {
  return
 }
  ...
}

```

调用 `Load` 方法检查 `m.read` 中是否存在这个元素。若存在，且没有被标记为删除状态，则尝试存储。

若该元素不存在或已经被标记为删除状态，则继续走到下面流程：

```
func (m *Map) Store(key, value interface{}) {
 ...
 m.mu.Lock()
 read, _ = m.read.Load().(readOnly)
 if e, ok := read.m[key]; ok {
  if e.unexpungeLocked() {
   m.dirty[key] = e
  }
  e.storeLocked(&value)
 } else if e, ok := m.dirty[key]; ok {
  e.storeLocked(&value)
 } else {
  if !read.amended {
   m.dirtyLocked()
   m.read.Store(readOnly{m: read.m, amended: true})
  }
  m.dirty[key] = newEntry(value)
 }
 m.mu.Unlock()
}

```

由于已经走到了 dirty 的流程，因此开头就直接调用了 `Lock` 方法**上互斥锁**，保证数据安全，也是凸显**性能变差的第一幕**。

其分为以下三个处理分支：

*   若发现 read 中存在该元素，但已经被标记为已删除（expunged），则说明 dirty 不等于 nil（dirty 中肯定不存在该元素）。其将会执行如下操作。
    

*   将元素状态从已删除（expunged）更改为 nil。
    
*   将元素插入 dirty 中。
    

*   若发现 read 中不存在该元素，但 dirty 中存在该元素，则直接写入更新 entry 的指向。
    
*   若发现 read 和 dirty 都不存在该元素，则从 read 中复制未被标记删除的数据，并向 dirty 中插入该元素，赋予元素值 entry 的指向。
    

我们理一理，写入过程的整体流程就是：

*   查 read，read 上没有，或者已标记删除状态。
    
*   上互斥锁（Mutex）。
    
*   操作 dirty，根据各种数据情况和状态进行处理。
    

回到最初的话题，为什么他写入性能差那么多。究其原因：

*   写入一定要会经过 read，无论如何都比别人多一层，后续还要查数据情况和状态，性能开销相较更大。
    
*   （第三个处理分支）当初始化或者 dirty 被提升后，会从 read 中复制全量的数据，若 read 中数据量大，则会影响性能。
    

可得知 `sync.Map` 类型不适合写多的场景，读多写少是比较好的。

若有大数据量的场景，则需要考虑 read 复制数据时的偶然性能抖动是否能够接受。

### 删除过程

这时候可能有小伙伴在想了。写入过程，理论上和删除不会差太远。怎么 `sync.Map` 类型的删除的性能似乎还行，这里面有什么猫腻？

源码如下：

```
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
 read, _ := m.read.Load().(readOnly)
 e, ok := read.m[key]
 ...
  if ok {
  return e.delete()
 }
}

```

删除是标准的开场，依然先到 read 检查该元素是否存在。

若存在，则调用 `delete` 标记为 expunged（删除状态），非常高效。可以明确在 read 中的元素，被删除，性能是非常好的。

若不存在，也就是走到 dirty 流程中：

```
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
 ...
 if !ok && read.amended {
  m.mu.Lock()
  read, _ = m.read.Load().(readOnly)
  e, ok = read.m[key]
  if !ok && read.amended {
   e, ok = m.dirty[key]
   delete(m.dirty, key)
   m.missLocked()
  }
  m.mu.Unlock()
 }
 ...
 return nil, false
}

```

若 read 中不存在该元素，dirty 不为空，read 与 dirty 不一致（利用 amended 判别），则表明要操作 dirty，上互斥锁。

再重复进行双重检查，若 read 仍然不存在该元素。则调用 delete 方法从 dirty 中标记该元素的删除。

需要注意，出现频率较高的 delete 方法：

```
func (e *entry) delete() (value interface{}, ok bool) {
 for {
  p := atomic.LoadPointer(&e.p)
  if p == nil || p == expunged {
   return nil, false
  }
  if atomic.CompareAndSwapPointer(&e.p, p, nil) {
   return *(*interface{})(p), true
  }
 }
}

```

该方法都是将 entry.p 置为 nil，并且标记为 expunged（删除状态），而**不是真真正正的删除**。

注：不要误用 `sync.Map`，前段时间从字节大佬分享的案例来看，他们将一个连接作为 key 放了进去，于是和这个连接相关的，例如：buffer 的内存就永远无法释放了...

总结
--

通过阅读本文，我们明确了 `sync.Map` 和原生 map +互斥锁/读写锁之间的性能情况。

标准库 `sync.Map` 虽说支持并发读写 map，但更适用于读多写少的场景，因为他写入的性能比较差，使用时要考虑清楚这一点。

另外我们针对 `sync.Map` 的性能差异，进行了深入的源码剖析，了解到了其背后快、慢的原因，实现了知其然知其所以然。

经常看到并发读写 map 导致致命错误，实在是令人忧心。大家觉得如果本文不错，欢迎分享给更多的 Go 爱好者 ：）

## 鼓励

若有任何疑问欢迎评论区反馈和交流，**最好的关系是互相成就**，各位的**点赞**就是[煎鱼](https://github.com/eddycjy)创作的最大动力，感谢支持。

> 文章持续更新，可以微信搜【脑子进煎鱼了】阅读，本文 **GitHub** [github.com/eddycjy/blog](https://github.com/eddycjy/blog) 已收录，欢迎 Star 催更。


参考
--

*   Package sync
    
*   踩了 Golang sync.Map 的一个坑
    
*   go19-examples/benchmark-for-map
    
*   通过实例深入理解sync.Map的工作原理
    