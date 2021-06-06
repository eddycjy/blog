---
title: "Go 面试官：Go interface 的一个 “坑” 及原理分析"
date: 2021-04-05T16:12:59+08:00
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

前几天在读者交流群里看到一位小伙伴，针对 interface 的使用有了比较大的疑惑。

无独有偶，我也在网上看到有小伙伴在 Go 面试的时候被问到了：


![来自网上博客的截图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36d7ecb0da9e4b32a493dedce6ebc691~tplv-k3u1fbpfcp-watermark.image)

今天特意分享出来让大家避开这个坑。

## 例子一

第一个例子，如下代码：


```golang
func main() {
    var v interface{}
    v = (*int)(nil)
    fmt.Println(v == nil)
}
```

你觉得输出结果是什么呢？

答案是：

```
false
```
为什么不是 `true`。明明都已经强行置为 `nil` 了。是不是 Go 编译器有问题？

## 例子二

第二个例子，如下代码：

```golang
func main() {
    var data *byte
    var in interface{}

    fmt.Println(data, data == nil)
    fmt.Println(in, in == nil)

    in = data
    fmt.Println(in, in == nil)
}
```

你觉得输出结果是什么呢？

答案是：

```
<nil> true
<nil> true
<nil> false
```

这可就更奇怪了，为什么刚刚声明出来的 `data` 和 `in` 变量，确实是输出结果是 `nil`，判断结果也是 `true`。

怎么把变量 `data` 一赋予给变量 `in`，世界就变了？输出结果依然是 `nil`，但判定却变成了 `false`。

和上面的第一个例子结果类似，真是神奇。

## 原因

interface 判断与想象中不一样的根本原因是，interface 并不是一个指针类型，虽然他看起来很像，以至于误导了不少人。

我们钻下去 interface，interface 共有两类数据结构：

![](https://imgkr2.cn-bj.ufileos.com/560dcab5-e436-4a2c-bfee-6eba6faee1d2.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=HF1wka8F9MTc%252FVCTrMjhASpeeE4%253D&Expires=1615900476)

- `runtime.eface` 结构体：表示不包含任何方法的空接口，也称为 empty interface。
- `runtime.iface` 结构体：表示包含方法的接口。

看看这两者相应的底层数据结构：

```golang
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```

你会发现 interface 不是单纯的值，而是**分为类型和值**。

所以传统认知的此 nil 并非彼 nil，**必须得类型和值同时都为 nil 的情况下，interface 的 nil 判断才会为 true**。

## 解决办法

与其说是解决方法，不如说是委婉的破局之道。在不改变类型的情况下，方法之一是利用反射（reflect），如下代码：

```golang
func main() {
    var data *byte
    var in interface{}

    in = data
    fmt.Println(IsNil(in))
}

func IsNil(i interface{}) bool {
    vi := reflect.ValueOf(i)
    if vi.Kind() == reflect.Ptr {
        return vi.IsNil()
    }
    return false
}
```

利用反射来做 nil 的值判断，在反射中会有针对 interface 类型的特殊处理，最终输出结果是：true，达到效果。

其他方法的话，就是改变原有的程序逻辑，例如：
- 对值进行 nil 判断，再返回给 interface 设置。
- 返回具体的值类型，而不是返回 interface。

## 总结

Go interface 是 Go 语言中最常用的类型之一，大家用惯了 `if err != nil` 就很容易顺手就踩进去了。

建议大家要多留个心眼，如果对 interface 想要有更进一步的了解，可以看看我的这篇深入解析的文章：《一文吃透 Go 语言解密之接口 interface》。
