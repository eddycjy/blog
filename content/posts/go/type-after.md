---
title: "为什么 Go 语言把类型放在后面？"
date: 2021-12-31T12:55:10+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

前段时间看到大家在吵一个话题，那就是 Go 语言的类型声明，抠知识抠的非常细了，就是为什么他要放在后面。

示例代码如下：

```golang
var a []string
var b []string
```

其实在早年 Go 官方估计已经被问烦了，写过一篇《[Go's Declaration Syntax](https://go.dev/blog/declaration-syntax "Go's Declaration Syntax")》来具体介绍和说明情况。

为此煎鱼将参考并结合这篇官方资料，带大家一起了解为什么 Go 如此的 “与众不同” ，为什么要把类型放在后面。

## 类型前置

在业内目前有不少知名语言，也采取的是在声明变量类型时，把类型定义在名字前面。像是 C、C++、C#、Java 等：

```c
int x;
int x = 100;
```

基本的格式定义：<data_type> <variable_list>;。

上面的声明是一个简单的例子，如果更复杂一些，Go 官方还给出了著名的函数指针的例子：

```c
int (*fp)(int a, int b);
```

更进一步，如果返回值也是个函数指针类型，就会变成：

```c
int (*(*fp)(int (*)(int, int), int))(int, int)
```

这已经很难看出来是个 fp 的声明了。

## 类型后置

前面所举例的类型前置的编程语言，很多都是 C 系列中的一者。类型后置的代表，分别有：Go、Rust、Scala、Kotlin 等。

其实在很多类型后置的编程语言种，会采取变量名+冒号+类型的方式出现。就像 Rust 一样：

```rust
let x: i32;
```

基本的格式定义：

```
x: int
p: pointer to int
a: array[3] of int
```

Go 官方参照了这类类型后置的设计，并且为了简洁，进一步去掉了冒号和一些关键字，变成：

```golang
var a []string
```

我们再看回前面 fp 的声明的例子：

```c
int (*(*fp)(int (*)(int, int), int))(int, int)
```

再对比 Go 语言中就变成了：

```golang
f func(func(int,int) int, int) func(int, int) int
```

两者一对比，Go 语言代码可读性确实更高一些。

## 思考

### 后置类别

在类型声明上，实际上分为：变量类型后置、函数返回值后置。两者共同构建了前置还是后置，总不能一个前置，一个后置吧，那得多么的难受。

上方 C 语言和 Go 语言函数指针的例子，所对比带来的代码可读性提高，其实本质上是由**函数返回值后置**所带来的。

和类型前置、后置没太多直接关系。

### 核心思想

在类型后置上来讲，Go 官方核心思想是：**这种声明方式（从左到右的风格）的一个优点是，当类型变得更加复杂时，它的效果非常好**（One merit of this left-to-right style is how well it works as the types become more complex）。

Go 的变量名总是在前，在人的代码阅读上可以**保持从左到右阅读**，不需要像 C 语言一样在一大堆声明中用技巧找变量名对应的类型。

![The Clockwise/Spiral Rule](https://files.mdnice.com/user/3610/b025d274-ae83-404c-8072-2776cd790708.png)

为此甚至有人写了篇 C 语言的顺时针读法《[The Clockwise/Spiral Rule](http://c-faq.com/decl/spiral.anderson.html "The Clockwise/Spiral Rule")》，有兴趣可以阅读。

如此一对比，Go 语言的类型后置在复杂场景下与 C 语言的对比确实更好一些。

### 其他因素

#### 类型推导

诸如在类型推导的形式上也会更直观：

```golang
func main() {
    var s1 := "脑子进煎鱼了"
    var s2 string
}
```

也是一个可读性提高的问题。

#### 类型和名字谁更重要

不同设计者对谁更重要的理解也不一样。是类型更重要，还是名字更重要呢？

有的人认为是类型，有的人认为是名字。这就真的是千人千面，众口难调了。

## C# 的后悔

我们看看其他语言，C# 设计组成员之一，其实在《[Sharp Regrets: Top 10 Worst C# Features](https://www.informit.com/articles/article.aspx?p=2425867 "Sharp Regrets: Top 10 Worst C# Features")》中的第五点表达了个人对类型前置、后置的设计教训。


![](https://files.mdnice.com/user/3610/6d5665b7-125f-4202-8742-c11c433566d7.png)

核心观点是：从编程和数学两方面来看，都有一个约定，即计算的结果在右侧表示，所以在类 C 语言中，类型在左侧是很奇怪的。

在设计时，C# 本来计划把类型注释放在右边。但考虑到类 C 语言，因此遵循了其他语言的惯例。


## 总结

实际上该问题的研讨，在 2021 年的现在，大部分 case 都一一被反驳了。类型后置也不是一个与众不同的设计，很多语言都是如此。但既然要讨论 Go 语言，那更多的是站在设计者的角度去考虑。

结合 Go 所提供的官方资料，在当年的目的更多的是为了**在遇到复杂类型定义时，能保持一定的代码可读性**。

当然，这不可否认肯定包含 Go 开发团队的主观意识。有兴趣的可以具体挖挖背后的信息。

如果是你，**你会把类型放在前面，还是后面呢，为什么**？