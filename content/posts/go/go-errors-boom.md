---
title: "生产环境遇到一个 Go 问题，整组人都懵逼了..."
date: 2021-07-07T12:43:49+08:00
images:
tags: 
  - go
---

大家好，我是煎鱼。

前段时间正在疯狂写代码的时候，突然有一个读者给我提了一个问题，让我有了一定的兴趣：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1011e66fe104d228dcaa2083b554fc8~tplv-k3u1fbpfcp-watermark.image)

我还是比较感兴趣的，因为是生产环境、有代码，且整组人都懵逼的问题。

在征求了小伙伴的意见后，今天分享出来，大家也思考一下原因，一起规避这个 “坑”。

## 案例一

代码示例如下：

```golang
type MyErr struct {
    Msg string
}

func main() {
    var e error
    e = GetErr()
    log.Println(e == nil)
}

func GetErr() *MyErr {
    return nil
}

func (m *MyErr) Error() string {
    return "脑子进煎鱼了"
}
```

请思考一下，**这段程序的输出结果是什么**？

该程序所调用的 `GetErr` 方法所返回的是 `nil`，而外部判断是 `e == nil`，因此最终的输出结果是 true，对吗？ 

输出结果如下：

```
2021/04/04 08:39:04 false
```

答案是：false。

## 案例二

代码示例如下：

```golang
type Base interface {
    do()
}

type App struct {
}

func main() {
    var base Base
    base = GetApp()
    
    log.Println(base)
    log.Println(base == nil)
}

func GetApp() *App {
    return nil
}
func (a *App) do() {}
```

请思考一下，**这段程序的输出结果是什么**？

该程序调用了 `GetApp` 方法，该方法返回的是 `nil`，因此其赋值的 `base` 也是 `nil`。因此判断 `base == nil` 的最终输出结果是 `<nil>` 和 `true`，对吗？

输出结果如下：

```
2021/04/04 08:59:00 <nil>
2021/04/04 08:59:00 false
```

答案是：`<nil>` 和 false。

## 为什么

为什么，这两段 Go 程序是怎么回事...也太反直觉了？其背后的原因本质上还是对 Go 语言中 interface 的基本原理的理解。

在案例一中，虽然 `GetErr` 方法确实是返回了 `nil`，返回的类型也是具体的 `*MyErr` 类型。但是其接收的变量却不是具体的结构类型，而是 `error` 类型：

```golang
var e error
e = GetErr()
```

在 Go 语言中， `error` 类型本质上是 interface：

```golang
type error interface {
    Error() string
}
```

因此兜兜转转又回到了 interface 类型的问题，interface 不是单纯的值，而是**分为类型和值**。

所以传统认知的此 nil 并非彼 nil，**必须得类型和值同时都为 nil 的情况下，interface 的 nil 判断才会为 true**。

在案例一中，结合代码逻辑，更符合场景的是：

```golang
var e *MyErr
e = GetErr()
log.Println(e == nil)
```

输出结果就会是 true。

在案例二中，也是一样的结果，原因也是 interface。不管是 `error` 接口（interface），还是自定义的接口，背后原理一致，自然也就结果一致了。

## 总结

今天这篇文章，相当于是《Go 面试题：Go interface 的一个 “坑” 及原理分析》的变形了，毕竟是生产环境的代码改造而来，更贴合真实的实际场景。

下意识的直觉有时候不是绝对正确的，我们要正确的理解 Go 语言中的那些知识点，才能更好地实现早下班的理想和愿景。

