---
title: "面试官：为什么 Go 的负载因子是 6.5？"
date: 2021-12-31T12:55:07+08:00
toc: true
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

最近我有一个朋友，在网上看到一个有趣的段子，引发了我一些兴趣。

如下图：

![](https://files.mdnice.com/user/3610/e81547d8-8de7-4310-a62f-e59ce4c0def2.png)

听说是在最后的闲聊、吹水、聊人生、乱扯环节了，不是在技术环节了，所以大家也不用太在意什么技术评估法则（别杠）。

煎鱼作为一名技术号主，看到这里的 6.5，就想给大家挖一挖，这到底是何物，和大家一同学习和增长知识！

## 6.5 是什么

实际上在 Go 语言中，就存在 6.5 这一概念，与 map 存在直接关系，因此我们需要先了解 map 的基本数据结构，再介绍 6.5 的背景和由来。

开始学习 6.5 吧！

### 了解 map 底层

我以前在写《[深入理解 Go map：初始化和访问元素](https://eddycjy.com/posts/go/map/2019-03-05-map-access/)》时有介绍过 map 的基础数据结构。

基本结构如下图：

![map 基本数据结构](https://files.mdnice.com/user/3610/8679a84f-0d21-485d-b55f-dc9d70d5ddd1.png)

其中重要的一个基本单位是 hmap：

```golang
type hmap struct {
	count     int
	flags     uint8
	B         uint8
	noverflow uint16
	hash0     uint32
	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer
	nevacuate  uintptr
	extra *mapextra
}

type mapextra struct {
	overflow    *[]*bmap
	oldoverflow *[]*bmap
	nextOverflow *bmap
}
```

- count：map 的大小，也就是 len() 的值，代指 map 中的键值对个数。
- flags：状态标识，主要是 goroutine 写入和扩容机制的相关状态控制。并发读写的判断条件之一就是该值。
- B：**桶，最大可容纳的元素数量，值为 负载因子（默认 6.5） * 2 ^ B，是 2 的指数**。
- noverflow：溢出桶的数量。
- hash0：哈希因子。
- buckets：保存当前桶数据的指针地址（指向一段连续的内存地址，主要存储键值对数据）。
- oldbuckets，保存旧桶的指针地址。
- nevacuate：迁移进度。
- extra：原有 buckets 满载后，会发生扩容动作，在 Go 的机制中使用了增量扩容，如下为细项：
    - overflow 为 hmap.buckets （当前）溢出桶的指针地址。
    - oldoverflow 为 hmap.oldbuckets （旧）溢出桶的指针地址。
    - nextOverflow 为空闲溢出桶的指针地址。

我们关注到 hmap 的 B 字段，其值就是 6.5，他就是我们在苦苦寻找的 6.5，但他又是什么呢？

### 什么是负载因子

B 值，这里就涉及到一个概念：**负载因子（load factor），用于衡量当前哈希表中空间占用率的核心指标**，也就是每个 bucket 桶存储的平均元素个数。

另外负载因子**与扩容、迁移**等重新散列（rehash）行为有直接关系：
- 在程序运行时，会不断地进行插入、删除等，会导致 bucket 不均，内存利用率低，需要迁移。
- 在程序运行时，出现负载因子过大，需要做扩容，解决 bucket 过大的问题。

负载因子是哈希表中的一个重要指标，在各种版本的哈希表实现中都有类似的东西，主要目的是**为了平衡 buckets 的存储空间大小和查找元素时的性能高低**。

在接触各种哈希表时都可以关注一下，做不同的对比，看看各家的考量。

## 为什么是 6.5

了解是什么后，我们进一步深挖。

为什么 Go 语言中哈希表的负载因子是 6.5，为什么不是 8 ，也不是 1。这里面有可靠的数据支撑吗？

### 测试报告

实际上这是 Go 官方的经过认真的测试得出的数字，一起来看看官方的这份测试报告。

报告中共包含 4 个关键指标，如下：

| loadFactor | %overflow  |  bytes/entry | hitprobe |    missprobe |
| --- | --- | --- | --- | --- |
|  4.00   |  2.13   |  20.77   |  3.00   |  4.00   |
|  4.50   |  4.05  |  17.30   |  3.25   |  4.50   |
|  5.00   |  6.85   | 14.77    | 3.50    |  5.00   |
|  5.50   |  10.55    | 12.94    | 3.75    | 5.50    |
|  6.00   |  15.27   | 11.67    | 4.00    | 6.00    |
|  6.50   |  20.90   | 10.79    | 4.25    | 6.50    |
|  7.00   |  27.14   | 10.15    | 4.50    | 7.00    |
|  7.50   |  34.03   | 9.73    | 4.75    | 7.50    |
|  8.00   |  41.10   | 9.40    |  5.00   | 8.00    |

- loadFactor：负载因子，也有叫装载因子。
- %overflow：溢出率，有溢出 bukcet 的百分比。
- bytes/entry：每对 key/elem 的开销字节数.
- hitprobe：查找一个存在的 key 时，要查找的平均个数。
- missprobe：查找一个不存在的 key 时，要查找的平均个数。

### 选择数值

结合测试报告一看，好家伙，不测不知道，一测吓一跳，有依据了。

Go 官方发现：**负载因子太大了，会有很多溢出的桶。太小了，就会浪费很多空间**（too large and we have lots of overflow buckets, too small and we waste a lot of space）。

![来自 Go 官方源码说明](https://files.mdnice.com/user/3610/93b87f8d-a0ad-4503-b4f9-e3cf040468df.png)

根据这份测试结果和讨论，Go 官方把 Go 中的 map 的负载因子硬编码为 6.5，这就是 6.5 的选择缘由。

这意味着在 Go 语言中，**当 B（bucket）平均每个存储的元素大于或等于 6.5 时，就会触发扩容行为**，这是作为我们用户对这个数值最近的接触。

## 总结

在今天这篇文章中，我们先快速了解了 Go 语言中 map 的基本数据结构和设计，这和我们要解释的问题紧密相关。

紧接着针对开头所提出的 6.5，进行了介绍和说明，这其实是 map 中的负载因子。其数值的确定来源于 Go 官方的测试。

为什么是 6.5，你懂了吗？

## 参考
- [src/runtime/map.go](https://golang.org/src/runtime/map.go)
- [深度解析golang map](https://juejin.cn/post/6954707500151078919)
- [golang中map底层B值的计算逻辑](https://zhuanlan.zhihu.com/p/366472077)