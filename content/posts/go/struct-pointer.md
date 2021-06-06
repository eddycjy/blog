---
title: "你知道 Go 结构体和结构体指针调用有什么区别吗？"
date: 2021-06-06T12:21:30+08:00
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

前几天在分享《Go 结构体是否可以比较，为什么？》时，有小伙伴提出了新的问题：

![来自文章评论区](https://image.eddycjy.com/a4d34c5312339b9909e482c18f0cdf4a.png)

虽然大家提问题的速度已经超出了本鱼写文章的速度...不过作为宠粉狂鱼，在此刻清明假期时还是写下了这篇文章。

我在网上冲浪时搜索了相关问题，发现 6 年前就有 Go 开发者有一模一样的疑问，真是困扰了一代又一代的小伙伴。

![来自 stackoverflow.com](https://image.eddycjy.com/dbe910917c86e103648e314f79896a81.png)


本期的男主角是《**Go 结构体和结构体指针调用有什么区别**》，希望对大家有所帮助，带来一些思考。

**请在此处默念自己心目中的答案**，再和煎鱼一同研讨一波 Go 的技术哲学。

## 结构体是什么

在 Go 语言中有个基本类型，开发者们称之为结构体（struct）。是 Go 语言中非常常用的，基本定义：

```golang
type struct_variable_type struct {
    member definition
    member definition
    ...
    member definition
}
```

简单示例：

```golang
package main

import "fmt"

type Vertex struct {
    Name1 string
    Name2 string
}

func main() {
    v := Vertex{"脑子进了", "煎鱼"}
    v.Name2 = "蒸鱼"
    fmt.Println(v.Name2)
}
```

输出结果：

```
蒸鱼
```

这部分属于基础知识，因此不再过多解释。如果看不懂，建议重学 Go 语言语法基础。

## 结构体和指针调用

讲解前置概要后，直接进入本文主题。如下例子：

```golang
type MyStruct struct {
    Name string
}

func (s MyStruct) SetName1(name string) {
    s.Name = name
}

func (s *MyStruct) SetName2(name string) {
    s.Name = name
}
```

该程序声明了一个 `User` 结构体，其包含两个结构体方法，分别是 `SetName1` 和 `SetName2` 方法，两者之间的差异就是**引用的方式不同**。

进一步延伸，这两者有什么区别，什么情况下用哪种，有没有什么注意事项？

注：很巧，我有一个朋友，当年刚上手 Go 语言时，就纠结过这个问题。

## 两者区别

从许多小伙伴的反馈来看，这两个例子之间的区别可能会让人感到困惑，经常会有人纠结要不要使用 “指针”，又担心 GC 什么的。

实际上情况没那么复杂，看看下面的例子：

```golang
func (s MyStruct) SetName1(name string) 
func (s *MyStruct) SetName2(name string)
```
当在一个类型上定义一个方法时，接收器（在上面的例子中是 s）的行为就像它是方法的一个参数一样。其相当于：

```golang
 func SetName1(s MyStruct, name string){
    u.Name = name
 }

 func SetName2(s *MyStruct,name string){
    u.Name = name
 }
```

因此结构体方法是要将接收器定义成值，还是指针。这本质上与函数参数应该是值还是指针是同一个问题。

## 如何选择

整体有以下几个考虑因素，按重要程度顺序排列：

1. 在使用上的考虑：方法是否需要修改接收器？如果需要，接收器必须是一个指针。

2. 在效率上的考虑：如果接收器很大，比如：一个大的结构体，使用指针接收器会好很多。

3. 在一致性上的考虑：如果类型的某些方法必须有指针接收器，那么其余的方法也应该有指针接收器，所以无论类型如何使用，方法集都是一致的。

回到上面的例子中，从功能使用角度来看：
- 如果 `SetName2` 方法修改了 s 的字段，调用者是可以看到这些字段值变更的，因为其是指针引用，本质上是同一份。
- 相对 `SetName1` 方法来讲，该方法是用调用者参数的副本来调用的，本质上是值传递，它所做的任何字段变更对调用者来说是看不见的。

另外对于基本类型、切片和小结构等类型，值接收器是非常廉价的。

因此除非方法的语义需要指针，那么值接收器是最高效和清晰的。在 GC 方面，也不需要过度关注。出现时再解决就好了。

## 总结

在本文中，我们针对 Go 结构体和结构体指针调用有什么区别，这个问题进行了深入浅出的分析和说明。

而在本文中所介绍的部分内容，来自于官方 FAQ 的 “Should I define methods on values or pointers?”，可以认为是官方给出的基本解答了（问的人是真的多）。

谁疑惑这个问题，转发这篇文章，吸就完了。