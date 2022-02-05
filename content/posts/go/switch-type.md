---
title: "Go 泛型玩出花，详解新提案 switch type！"
date: 2021-12-31T12:55:23+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

前面写过好几篇 Go 泛型的语法、案例介绍，新的一手 Go 消息。实际上，随着一些提案被接受，新的提案也逐渐冒出。

这不，我发现有了泛型后，大家可以更进一步玩出花来了。看到了一个 ”新“ 提案《[proposal: spec: generics: type switch on parametric types](https://github.com/golang/go/issues/45380 "proposal: spec: generics: type switch on parametric types")》，讲的就是增加泛型后的参数类型上的类型开关诉求。

跟着煎鱼一起掌握新的 Go 知识吧！

## 新提案

新的提案内容是希望增加一个新的变种语句，允许在 switch 语句中使用泛型时，能够进一步便捷的约束其类型参数。

例如：

```go
switch type T {
case A1:
case A2, A3:
   ...
}
```

也就是 switch-type 语句的 **T 类型可以是一个泛型的类型参**，case 所对应的的类型可以是任何类型，包括泛型的约束类型。

假设类型 T 的类型有可能是以下：

```go
interface{
    C
    A
}
```

可以借助泛型的近似元素来约束：

```go
    interface{
        C
        A1 | A2 | ... | An
    }
```

甚至还可以在 case 上有新的写法：

```go
case interface {~T}:
```

在支持泛型后，**switch 在 type 和 case 上会存在很多种可能性**，需要进行具体的特性支持，这个提案就是为此出现。

## 实际案例

### 案例一：多类型元素

```go
type Stringish interface {
	string | fmt.Stringer
}

func Concat[S Stringish](x []S "S Stringish") string {
    switch type S {
    case string:
        ...
    case fmt.Stringer:
        ...
    }
 }
```

类型 S 能够支持 string 和 fmt.Stringer 类型，case 配套对应实现。

### 案例二：近似元素

```go
type Constraint interface {
    ~int | ~int8 | ~string
}

func ThisSyntax[T Constraint]( "T Constraint") {
    switch type T {
    case ~int | ~int8:
        ...
    case ~string:
        ...
    }
}

func IsClearerThanThisSyntax[T Constraint]( "T Constraint") {
    switch type T {
    case interface{~int | ~int8 }:
        ...
    case interface{ ~string }:
        ...
    }
}
```

类型 T 可能有很多类型，程序中用到了近似元素，也就是基础类型是 int、int8、string，这些类型中的任何一种都能够满足这个约束。

为此，switch-type 支持了，case 也要配套支持该特性。

## 争议点

看到这里可能大家也想到了，这个味道很似曾相识，好像某个语法能够支持。因此，这个提案下最有争议的，就是与原有的类型断言的重复。

原有的类型断言如下：

```go
switch T.(type) {
case string:
   ...
default:
   ...
}
```

新的类型判别如下：

```go
switch type T {
case A1:
case A2, A3:
   ...
}
```

这么咋一看，其实类型断言的完全可以取代新的，那岂不是重复建设，造轮子了？

其实是没有完全取代的。差异点如下：

```go
type ApproxString interface { ~string }

func F[T ApproxString](v T "T ApproxString") {
    switch (interface{})(v).(type) {
    case string:
        fmt.Println(v)
    default:
        panic("脑子没进煎鱼")
    }
}

type MyString string

func main() {
    F(MyString("脑子进煎鱼了"))
}
```

看出来差别在哪了吗，答案是什么？


答案是：会抛出恐慌（panic）。

你可能纠结了，问题出在哪里？这传入的 ”脑子进煎鱼了“ 的类型是 `MyString`，他的基础类型是 `string` 类型，也满足 `ApproxString` 类型的近似类型 `~string` 的要求，怎么就不行了...

根本原因是因为他的类型是 interface，而非 string 类型。所以走到了 defalut 分支的恐慌。


## 总结

今天给大家介绍了 Go 泛型的最新消息，在上一个提案被合并后，该提案也有一些新的动静。不过 Go 官方表态，会等熟练掌握泛型实践后，再继续推动该提案。

我相信，原有的 `switch.(type)` 和 `switch type` 很大概率在 Go 底层会变成同一个逻辑块处理，再逐渐过渡。

这个提案的目的还是**为了解决若干引入泛型后，所带入的 BUG/需求**，正正是需要新的语法结构来解决的。

**你对此有什么看法呢**，欢迎在评论区留言和交流：）