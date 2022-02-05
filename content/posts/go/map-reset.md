---
title: "Go map 如何缩容？"
date: 2021-12-31T12:55:07+08:00
toc: true
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

前几天看到 Go 圈子的著名股神（不是我...），在归类中简单的提到了 Go 语言中 map 的缩容的描述，这让我对其产生了兴趣，想要来一探究竟。

我们常常喊扩缩容，扩缩容，但社区里都是清一色分析扩容机制，Go 面试官也都是卷 Go 语言 map 的扩容机制...

在 **Go 语言中的 map 缩容机制是怎么做的**呢，今天就由煎鱼带大家一起研讨围观一轮。

## 基本分析

在 Go 底层源码 src/runtime/map.go 中，扩缩容的处理方法是 grow 为前缀的方法来处理的。

其中扩缩容涉及到的是插入元素的操作，对应 mapassign 方法：

```golang
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  ...
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again
	}
  ...
}

func (h *hmap) growing() bool {
	return h.oldbuckets != nil
}

func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	if B > 15 {
		B = 15
	}
  
	return noverflow >= uint16(1)<<(B&15)
}
```

核心看到针对扩缩容的判断逻辑：
- 当前没有在扩容：条件为 oldbuckets 不为 nil。
- 是否可以进行扩容：条件为 `hmap.count`> hash 桶数量 `(2^B)*6.5`。其中 `hmap.count` 指的是map 的数据数目， `2^B` 仅指 hash 数组的大小，不包含溢出桶。
- 是否可以进行缩容：条件为溢出桶（noverflow）的数量 >= 32768（1<<15）。

而我们可以关注到，无论是扩容还是缩容，其都是由 `hashGrow` 方法进行处理：

```golang
func hashGrow(t *maptype, h *hmap) {
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
  ...
}
```

若是扩容，则 bigger 为 1，也就是 B+1。代表 hash 表容量扩大 1 倍。不满足就是缩容，也就是 hash 表容量不变。

可以得出结论：map 的**扩缩容的主要区别在于 hmap.B 的容量大小改变**。而缩容由于 hmap.B 压根没变，内存空间的占用也是没有变化的。

## 带来的隐患

这种方式其实是存在运行隐患的，也就是**导致在删除元素时，并不会释放内存，使得分配的总内存不断增加**。如果一个不小心，拿 map 来做大 key/value 的存储，也不注意管理，很容易就内存爆了。

也就是 Go 语言的 map 目前实现的是 ”伪缩容“，仅针对溢出桶过多的情况。若是触发缩容，hash 数组的占用的内存大小不变。

若要实现 ”真缩容“，Go Contributor @josharian 表示目前**唯一可用的解决方法是：创建一个新的 map 并从旧的 map 中复制元素**。

示例如下：

```golang
old := make(map[int]int, 9999999)
new := make(map[int]int, len(old))
for k, v := range old {
    new[k] = v
}
old = new
...
```

无比的像，复制粘贴，大的往小的挪动，再删掉大的就可以了。

果然程序员的解决方案都是相似的。

## 为什么不支持

下述内容会主要基于如下两个 issues 和 proposal 来分析：
1. 《[runtime: shrink map as elements are deleted](https://github.com/golang/go/issues/20135 "runtime: shrink map as elements are deleted")》
2. 《[proposal: runtime: add way to clear and reuse a map's working storage](https://github.com/golang/go/issues/45328 "proposal: runtime: add way to clear and reuse a map's working storage")》

目前 map 的缩容处理起来比较棘手，最早的 issues 是 2016 年提出的，也有人提过一些提案，但都因为种种原因被拒绝了。

简单来讲，就是没有找到一个很好的方法实现，存在明确的实现成本问题，没法很方便的 ”告诉“ Go 运行时，我要：
1. 记得保留存储空间，我要立即重用 map。
2. 赶紧释放存储空间，map 从现在开始会小很多。

抽象来看症结是：**需要保证增长结果在下一个开始之前完成**，此处的增长指的是 ”从小到大，从一个大小到相同大小，从大到小“ 的复杂过程。

这属于一个多重 case，从而导致也就一直拖着，慢慢想。

## 总结

Go 语言中 map 的扩容机制是大家经常思考和学习的，但是缩容方面现在也是一个大 ”坑“。虽然不误用就没问题。

但是一旦新同学来了，不知道，一塞，就会出问题。我有一个朋友，之前面试时，就听闻有人会塞几个 GB 的数据进 map，可想而知还是很危险的。

现有 map 的缩容机制，是存在短板的，背后有着比较多重的纠结，不知道你**有没有什么好的建议或想法呢，欢迎大家一起来讨论**！