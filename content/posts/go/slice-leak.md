---
title: "Go 切片导致内存泄露，被坑两次了！"
date: 2021-12-31T12:55:09+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

前段时间在我的 Go 读者群里，有小伙伴们在纠结切片（slice）的问题，我写了这篇《[Go 切片这道题，吵了一个下午！](https://mp.weixin.qq.com/s/kEQI74ge6VhvNEr1d3JW-Q)》，引起了一拨各种讨论，还是比较欣慰的。

这不，有小伙伴给我提出了新的题材：

![来自读者微信提问](https://files.mdnice.com/user/3610/a97e5e6d-b58d-44f2-b858-b9ca12780180.png)

提出的是 Go 中很容易踩坑的切片内存泄露问题。作为宠粉的煎鱼肯定不会放过，争取让大家都避开这个 “坑”。

今天这篇文章，就由煎鱼带大家来了解这个问题：Go 切片可能可以怎么泄露法？

## 切片泄露的可能

在业务代码的编写上，我们经常会接受来自外部的接口数据，再把他插入到对应的数据结构中去，再进行下一步的业务聚合、裁剪、封装、处理。

像在 PHP 语言，常常会放到数组（array）中。在 Go 语言，会放到切片（slice）中。因此在 Go 的切片处理逻辑中，常常会涉及到如下类似的动作。

示例代码如下：

```golang
var a []int

func f(b []int) []int {
	a = b[:2]
	return a
}

func main() {
    ...
}
```

仔细想想，**这段程序有没有问题**，是否存在内存泄露的风险？

答案是：有的。有明确的切片内存泄露的可能性和风险。

## 切片底层结构

可能有些小伙伴会疑惑，怎么就有问题了，是哪里有问题？

这里就得复习一下切片的底层基本数据结构了，切片在运行时的表现是 SliceHeader 结构体，定义如下：

```golang
type SliceHeader struct {
 Data uintptr
 Len  int
 Cap  int
}
```
- Data：指向具体的底层数组。
- Len：代表切片的长度。
- Cap：代表切片的容量。

要点是：切片真正存储数据的地方，是一个数组。切片的 Data 属性中**存储的是指向所引用的数组指针地址**。

## 背后的原因

在上述案例中，我们有一个包全局变量 a，共有 2 个切片 a 和 b，截取了 b 的一部分赋值给了 a，两者存在着关联。

从程序的直面来看，截取了 b 的一部分赋值给了 a，结构似乎是如下图：

![](https://files.mdnice.com/user/3610/856aff0a-14bb-4dca-9324-4b852f25dd12.png)

但我们进一步打开程序底层来看，他应该是如下图所示：

![](https://files.mdnice.com/user/3610/c79008dd-771d-475b-8c41-6b654216feff.png)

切片 a 和 b 都共享着同一个底层数组（共享内存块），sliceB 包含全部所引用的字符。sliceA 只包含了 [:2]，也就是 0 和 1 两个索引位的字符。

那他们泄露在哪里了？

## 泄露的点

泄露的点，就在于虽然切片 b 已经在函数内结束了他的使命了，不再使用了。但切片 a 还在使用，切片 a 和 切片 b 引用的是同一块底层数组（共享内存块）。

关键点：**切片 a 引用了底层数组中的一段**。

![](https://files.mdnice.com/user/3610/0a4353e0-e793-41b5-a2dc-a6a25a39a519.png)

虽然切片 a 只有底层数组中 0 和 1 两个索引位正在被使用，其余未使用的底层数组空间毫无作用。但由于正在被引用，他们也不会被 GC，因此造成了泄露。

## 解决办法

解决的办法，就是利用切片的特性。当切片的容量空间不足时，会**重新申请一个新的底层数组来存储，让两者彻底分手**。

示例代码如下：

```golang
var a []int
var c []int    // 第三者

func f(b []int) []int {
	a = b[:2]
  
  // 新的切片 append 导致切片扩容
	c = append(c, b[:2]...)
	fmt.Printf("a: %p\nc: %p\nb: %p\n", &a[0], &c[0], &b[0])
  
	return a
}
```

输出结果：

```
a: 0xc000102060
c: 0xc000124010
b: 0xc000102060
```

这段程序，新增了一个变量 c，他容量为 0。此时将期望的数据，追加过去。自然而然他就会遇到容量空间不足的情况，也就能实现申请新底层数据。

我们再将原本的切片置为 nil，就能成功实现两者分手的目标了。

## 总结

在今天这篇文章中，我们介绍了 Go 切片的一种常见的内存泄露方式。虽然我们在日常使用的时候可能没关注到。

主要原因还是由于切片的大多数使用场景，体量都比较小。又或是不知不觉就自己扩容了，就变成暂时性泄露了。

这依然是存在风险的，在编写 Go 代码时需要谨慎。毕竟这可是 **Go 语言官方自己都踩过坑的 “坑”**。

## 参考
- [An interesting way to leak memory with Go slices](https://utcc.utoronto.ca/~cks/space/blog/programming/GoSlicesMemoryLeak)
- [internal/poll: avoid memory leak in Writev](https://github.com/golang/go/pull/32138/files)
- [slice 类型内存泄露的逻辑](https://xargin.com/logic-of-slice-memory-leak/)
- [golang slice内存泄露回收](https://zhuanlan.zhihu.com/p/149381458)