---

title:      "为什么遍历 Go map 是无序的"
date:       2019-04-07 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - 源码分析
---

![image](http://wx2.sinaimg.cn/large/006fVPCvly1g1s1ah84k8j30k70dvaac.jpg)

有的小伙伴没留意过 Go map 输出顺序，以为它是稳定的有序的；有的小伙伴知道是无序的，但却不知道为什么？有的却理解错误？今天我们将通过本文，揭开 `for range map` 的 “神秘” 面纱，看看它内部实现到底是怎么样的，输出顺序到底是怎么样？

## 前言

```go
func main() {
	m := make(map[int32]string)
	m[0] = "EDDYCJY1"
	m[1] = "EDDYCJY2"
	m[2] = "EDDYCJY3"
	m[3] = "EDDYCJY4"
	m[4] = "EDDYCJY5"

	for k, v := range m {
		log.Printf("k: %v, v: %v", k, v)
	}
}
```

假设运行这段代码，输出结果是按顺序？还是无序输出呢？

```
2019/04/03 23:27:29 k: 3, v: EDDYCJY4
2019/04/03 23:27:29 k: 4, v: EDDYCJY5
2019/04/03 23:27:29 k: 0, v: EDDYCJY1
2019/04/03 23:27:29 k: 1, v: EDDYCJY2
2019/04/03 23:27:29 k: 2, v: EDDYCJY3
```

从输出结果上来讲，是非固定顺序输出的，也就是每次都不一样（标题也讲了）。但这是为什么呢？

首先**建议你先自己想想原因**。其次我在面试时听过一些说法。有人说因为是哈希的所以就是无（乱）序等等说法。当时我是有点 ？？？

这也是这篇文章出现的原因，希望大家可以一起研讨一下，理清这个问题 ：）

## 看一下汇编

```
    ...
	0x009b 00155 (main.go:11)	LEAQ	type.map[int32]string(SB), AX
	0x00a2 00162 (main.go:11)	PCDATA	$2, $0
	0x00a2 00162 (main.go:11)	MOVQ	AX, (SP)
	0x00a6 00166 (main.go:11)	PCDATA	$2, $2
	0x00a6 00166 (main.go:11)	LEAQ	""..autotmp_3+24(SP), AX
	0x00ab 00171 (main.go:11)	PCDATA	$2, $0
	0x00ab 00171 (main.go:11)	MOVQ	AX, 8(SP)
	0x00b0 00176 (main.go:11)	PCDATA	$2, $2
	0x00b0 00176 (main.go:11)	LEAQ	""..autotmp_2+72(SP), AX
	0x00b5 00181 (main.go:11)	PCDATA	$2, $0
	0x00b5 00181 (main.go:11)	MOVQ	AX, 16(SP)
	0x00ba 00186 (main.go:11)	CALL	runtime.mapiterinit(SB)
	0x00bf 00191 (main.go:11)	JMP	207
	0x00c1 00193 (main.go:11)	PCDATA	$2, $2
	0x00c1 00193 (main.go:11)	LEAQ	""..autotmp_2+72(SP), AX
	0x00c6 00198 (main.go:11)	PCDATA	$2, $0
	0x00c6 00198 (main.go:11)	MOVQ	AX, (SP)
	0x00ca 00202 (main.go:11)	CALL	runtime.mapiternext(SB)
	0x00cf 00207 (main.go:11)	CMPQ	""..autotmp_2+72(SP), $0
	0x00d5 00213 (main.go:11)	JNE	193
	...
```

我们大致看一下整体过程，重点处理 Go map 循环迭代的是两个 runtime 方法，如下：

- runtime.mapiterinit
- runtime.mapiternext

但你可能会想，明明用的是 `for range` 进行循环迭代，怎么出现了这两个函数，怎么回事？

## 看一下转换后

```go
var hiter map_iteration_struct
for mapiterinit(type, range, &hiter); hiter.key != nil; mapiternext(&hiter) {
    index_temp = *hiter.key
    value_temp = *hiter.val
    index = index_temp
    value = value_temp
    original body
}
```

实际上编译器对于 slice 和 map 的循环迭代有不同的实现方式，并不是 `for` 一扔就完事了，还做了一些附加动作进行处理。而上述代码就是 `for range map` 在编译器展开后的伪实现

## 看一下源码

### runtime.mapiterinit

```go
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	...
	it.t = t
	it.h = h
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.kind&kindNoPointers != 0 {
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))
	it.bucket = it.startBucket
    ...

	mapiternext(it)
}
```

通过对 `mapiterinit` 方法阅读，可得知其主要用途是在 map 进行遍历迭代时**进行初始化动作**。共有三个形参，用于读取当前哈希表的类型信息、当前哈希表的存储信息和当前遍历迭代的数据

#### 为什么

咱们关注到源码中 `fastrand` 的部分，这个方法名，是不是迷之眼熟。没错，它是一个生成随机数的方法。再看看上下文：

```go
...
// decide where to start
r := uintptr(fastrand())
if h.B > 31-bucketCntBits {
	r += uintptr(fastrand()) << 31
}
it.startBucket = r & bucketMask(h.B)
it.offset = uint8(r >> h.B & (bucketCnt - 1))

// iterator state
it.bucket = it.startBucket
```

在这段代码中，它生成了随机数。用于决定从哪里开始循环迭代。更具体的话就是根据随机数，选择一个桶位置作为起始点进行遍历迭代

因此每次重新 `for range map`，你见到的结果都是不一样的。那是因为它的起始位置根本就不固定！

### runtime.mapiternext

```go
func mapiternext(it *hiter) {
    ...
    for ; i < bucketCnt; i++ {
		...
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.valuesize))
		...
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey || alg.equal(k, k)) {
			...
			it.key = k
			it.value = v
		} else {
			rk, rv := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.value = rv
		}
		it.bucket = bucket
		if it.bptr != b {
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

在上小节中，咱们已经选定了起始桶的位置。接下来就是通过 `mapiternext` 进行**具体的循环遍历动作**。该方法主要涉及如下：

- 从已选定的桶中开始进行遍历，寻找桶中的下一个元素进行处理
- 如果桶已经遍历完，则对溢出桶 `overflow buckets` 进行遍历处理

通过对本方法的阅读，可得知其对 buckets 的**遍历规则**以及对于扩容的一些处理（这不是本文重点。因此没有具体展开）

## 总结

在本文开始，咱们先提出核心讨论点：“为什么 Go map 遍历输出是不固定顺序？”。而通过这一番分析，原因也很简单明了。就是 `for range map` 在开始处理循环逻辑的时候，就做了随机播种...

你想问为什么要这么做？当然是官方有意为之，因为 Go 在早期（1.0）的时候，虽是稳定迭代的，但从结果来讲，其实是无法保证每个 Go 版本迭代遍历规则都是一样的。而这将会导致可移植性问题。因此，改之。也请不要依赖...

## 参考

- [Go maps in action](https://blog.golang.org/go-maps-in-action)
