---

title:      "Go Slice 最大容量大小是怎么来的"
date:       2019-01-06 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - 源码分析
---

![image](https://s2.ax1x.com/2020/02/27/3wnHRU.png)

## 前言

在《深入理解 Go Slice》中，我们提到了 “根据其类型大小去获取能够申请的最大容量大小” 的处理逻辑。今天我们将更深入地去探究一下，底层到底做了什么东西，涉及什么知识点？

Go Slice 对应代码如下：

```go
func makeslice(et *_type, len, cap int) slice {
	maxElements := maxSliceCap(et.size)
	if len < 0 || uintptr(len) > maxElements {
		...
	}

	if cap < len || uintptr(cap) > maxElements {
		...
	}

	p := mallocgc(et.size*uintptr(cap), et, true)
	return slice{p, len, cap}
}
```

根据想要追寻的逻辑，定位到了 `maxSliceCap` 方法，它会根据**当前类型的大小获取到了所允许的最大容量大小**来进行阈值判断，也就是安全检查。这是浅层的了解，我们继续追下去看看还做了些什么？

## maxSliceCap

```go
func maxSliceCap(elemsize uintptr) uintptr {
	if elemsize < uintptr(len(maxElems)) {
		return maxElems[elemsize]
	}
	return maxAlloc / elemsize
}
```

## maxElems

```go
var maxElems = [...]uintptr{
	^uintptr(0),
	maxAlloc / 1, maxAlloc / 2, maxAlloc / 3, maxAlloc / 4,
	maxAlloc / 5, maxAlloc / 6, maxAlloc / 7, maxAlloc / 8,
	maxAlloc / 9, maxAlloc / 10, maxAlloc / 11, maxAlloc / 12,
	maxAlloc / 13, maxAlloc / 14, maxAlloc / 15, maxAlloc / 16,
	maxAlloc / 17, maxAlloc / 18, maxAlloc / 19, maxAlloc / 20,
	maxAlloc / 21, maxAlloc / 22, maxAlloc / 23, maxAlloc / 24,
	maxAlloc / 25, maxAlloc / 26, maxAlloc / 27, maxAlloc / 28,
	maxAlloc / 29, maxAlloc / 30, maxAlloc / 31, maxAlloc / 32,
}
```

`maxElems` 是包含一些预定义的切片最大容量值的查找表，索引是切片元素的类型大小。而值看起来 “奇奇怪怪” 不大眼熟，都是些什么呢。主要是以下三个核心点：

- ^uintptr(0)
- maxAlloc
- maxAlloc / typeSize

### ^uintptr(0)

```go
func main() {
	log.Printf("uintptr: %v\n", uintptr(0))
	log.Printf("^uintptr: %v\n", ^uintptr(0))
}
```

输出结果：

```
2019/01/05 17:51:52 uintptr: 0
2019/01/05 17:51:52 ^uintptr: 18446744073709551615
```

我们留意一下输出结果，比较神奇。取反之后为什么是 18446744073709551615 呢？

### uintptr 是什么

在分析之前，我们要知道 uintptr 的本质（真面目），也就是它的类型是什么，如下：

```go
type uintptr uintptr
```

uintptr 的类型是自定义类型，接着找它的真面目，如下：

```
#ifdef _64BIT
typedef	uint64		uintptr;
#else
typedef	uint32		uintptr;
#endif
```

通过对以上代码的分析，可得出以下结论：

- 在 32 位系统下，uintptr 为 uint32 类型，占用大小为 4 个字节
- 在 64 位系统下，uintptr 为 uint64 类型，占用大小为 8 个字节

### ^uintptr 做了什么事

^ 位运算符的作用是**按位异或**，如下：

```go
func main() {
	log.Println(^1)
	log.Println(^uint64(0))
}
```

输出结果：

```
2019/01/05 20:44:49 -2
2019/01/05 20:44:49 18446744073709551615
```

接下来我们分析一下，这两段代码都做了什么事情呢

#### ^1

二进制：0001

按位取反：1110

该数为有符号整数，最高位为符号位。低三位为表示数值。按位取反后为 1110，根据先前的说明，最高位为 1，因此表示为 -。取反后 110 对应十进制 -2

#### ^uint64(0)

二进制：0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000

按位取反：1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111

该数为无符号整数，该位取反后得到十进制值为：18446744073709551615

这个值是不是看起来很眼熟呢？没错，就是 `^uintptr(0)` 的值。也印证了其底层数据类型为 uint64 的事实 （本机为 64 位）。同时它又代表如下：

- math.MaxUint64
- 2 的 64 次方减 1

### maxAlloc

```go
const GoarchMips = 0
const GoarchMipsle = 0
const GoarchWasm = 0

...

_64bit = 1 << (^uintptr(0) >> 63) / 2

heapAddrBits = (_64bit*(1-sys.GoarchWasm))*48 + (1-_64bit+sys.GoarchWasm)*(32-(sys.GoarchMips+sys.GoarchMipsle))

maxAlloc = (1 << heapAddrBits) - (1-_64bit)*1
```

`maxAlloc` 是**允许用户分配的最大虚拟内存空间**。在 64 位，理论上可分配最大 `1 << heapAddrBits` 字节。在 32 位，最大可分配小于 `1 << 32` 字节

在本文，仅需了解它承载的是什么就好了。具体的在以后内存管理的文章再讲述

注：该变量在 go 10.1 为 `_MaxMem`，go 11.4 已改为 `maxAlloc`。相关的 `heapAddrBits` 计算方式也有所改变

### maxAlloc / typeSize

我们再次回顾 `maxSliceCap` 的逻辑代码，这次重点放在控制逻辑，如下：

```go
// func makeslice
maxElements := maxSliceCap(et.size)

...

// func maxSliceCap
if elemsize < uintptr(len(maxElems)) {
	return maxElems[elemsize]
}
return maxAlloc / elemsize
```

通过这段代码和 Slice 上下文逻辑，可得知在想得到该类型的最大容量大小时。会根据对应的类型大小去查找表查找索引（索引为类型大小，摆放顺序是有考虑原因的）。“迫不得已的情况下” 才会手动的计算它的值，最终计算得到的内存字节大小都为该类型大小的整数倍

查找表的设置，更像是一个优化逻辑。减少常用的计算开销 :)

## 总结

通过本文的分析，可得出 Slice 所允许申请的最大容量大小，与当前**值类型**和当前**平台位数**有直接关系

## 最后

本文与[《有点不安全却又一亮的 Go unsafe.Pointer》](https://github.com/EDDYCJY/blog/blob/master/golang/pkg/2018-12-15-%E6%9C%89%E7%82%B9%E4%B8%8D%E5%AE%89%E5%85%A8%E5%8D%B4%E5%8F%88%E4%B8%80%E4%BA%AE%E7%9A%84Go-unsafe-Pointer.md)一同属于[《深入理解 Go Slice》](https://github.com/EDDYCJY/blog/blob/master/golang/pkg/2018-12-11-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Go-Slice.md)的关联章节。如果你在阅读源码时，对这些片段有疑惑。记得想尽办法深究下去，搞懂它

短短的一句话其实蕴含着不少知识点，希望这篇文章恰恰好可以帮你解惑

注：本文 Go 代码基于版本 11.4
