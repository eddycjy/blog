---
title: "手撕 Go 面试官：Go 结构体是否可以比较，为什么？"
date: 2021-04-05T16:16:00+08:00
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

最近金三银四，是面试的季节。在我的 Go 读者交流群里出现了许多小伙伴在讨论自己面试过程中所遇到的一些 Go 面试题。

今天的男主角，是 Go 工程师的必修技能，也是极容易踩坑的地方，就是 “**Go 面试题：Go 结构体（struct）是否可以比较？**”

如果可以比较，是为什么？如果不可以比较，又是为什么？

**请在此处默念自己心目中的答案**，再往和煎鱼一起研讨一波 Go 的技术哲学。

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

## 比较两下

### 例子一

接下来正式开始研讨 Go 结构体比较的问题，第一个例子如下：

```golang
type Value struct {
    Name   string
    Gender string
}

func main() {
    v1 := Value{Name: "煎鱼", Gender: "男"}
    v2 := Value{Name: "煎鱼", Gender: "男"}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}
```

我们声明了两个变量，分别是 v1 和 v2。其都是 `Value` 结构体的实例化，是同一个结构体的两个实例。

他们的比较结果是什么呢，是输出 ”脑子进煎鱼了“，还是 ”脑子没进煎鱼“？

输出结果：

```
脑子进煎鱼了
```

最终输出结果是 ”脑子进煎鱼了“，初步的结论是可以结构体间比较的。皆大欢喜，那这篇文章是不是就要结束了？

当然不是...很多人都会踩到这个 Go 语言的坑，**真实情况是结构体是可比较，也不可比较的**，不要误入歧途了，这是一个非常 "有趣" 的现象。

### 例子二

接下来继续改造上面的例子，我们在原本的结构体中增加了指针类型的引用。

第二个例子如下：

```golang
type Value struct {
    Name   string
    Gender *string
}

func main() {
    v1 := Value{Name: "煎鱼", Gender: new(string)}
    v2 := Value{Name: "煎鱼", Gender: new(string)}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}
```

这段程序输出结果是什么呢，我们猜测一下，变量依然是同一结构体的两个实例，值的赋值方式和内容都是一样的，是否应当输出 “脑子进煎鱼了”？

答案是：脑子没进煎鱼。

### 例子三

我们继续不信邪，试试另外的基本类型，看看结果是不是还是相等的。

第三个例子如下：

```golang
type Value struct {
    Name   string
    GoodAt []string
}

func main() {
    v1 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    v2 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}
```

这段程序输出结果是什么呢？

答案是：

```
# command-line-arguments
./main.go:15:8: invalid operation: v1 == v2 (struct containing []string cannot be compared)
```
程序运行就直接报错，IDE 也提示错误，一只煎鱼都没能输出出来。

### 例子四

那不同结构体，相同的值内容呢，能否进行比较？

第四个例子：

```golang
type Value1 struct {
    Name string
}

type Value2 struct {
    Name string
}

func main() {
    v1 := Value1{Name: "煎鱼"}
    v2 := Value2{Name: "煎鱼"}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}
```

显然，会直接报错：

```
# command-line-arguments
./main.go:18:8: invalid operation: v1 == v2 (mismatched types Value1 and Value2)
```

那是不是就完全没法比较了呢？并不，我们可以借助强制转换来实现：

```golang
	if v1 == Value1(v2) {
		fmt.Println("脑子进煎鱼了")
		return
	}
```

这样程序就会正常运行，且输出 “脑子进煎鱼了”。当然，若是不可比较类型，依然是不行的。

## 为什么

为什么 Go 结构体有的比较就是正常，有的就不行，甚至还直接报错了。难道是有什么 “潜规则” 吗？

在 Go 语言中，Go 结构体有时候并不能直接比较，当其基本类型包含：slice、map、function 时，是不能比较的。若强行比较，就会导致出现例子中的直接报错的情况。

而指针引用，其虽然都是 `new(string)`，从表象来看是一个东西，但其具体返回的地址是不一样的。

因此若要比较，则需改为：

```golang
func main() {
    gender := new(string)
    v1 := Value{Name: "煎鱼", Gender: gender}
    v2 := Value{Name: "煎鱼", Gender: gender}
    ...
}
```

这样就可以保证两者的比较。如果我们被迫无奈，被要求一定要用结构体比较怎么办？

这时候可以使用反射方法 `reflect.DeepEqual`，如下：

```golang
func main() {
    v1 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    v2 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    if reflect.DeepEqual(v1, v2) {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}
```

这样子就能够正确的比较，输出结果为 “脑子进煎鱼了”。

例子中所用到的反射比较方法 `reflect.DeepEqual` 常用于判定两个值是否深度一致，其规则如下：

- 相同类型的值是深度相等的，不同类型的值永远不会深度相等。
- 当数组值（array）的对应元素深度相等时，数组值是深度相等的。
- 当结构体（struct）值如果其对应的字段（包括导出和未导出的字段）都是深度相等的，则该值是深度相等的。
- 当函数（func）值如果都是零，则是深度相等；否则就不是深度相等。
- 当接口（interface）值如果持有深度相等的具体值，则深度相等。
- ...

更具体的大家可到 `golang.org/pkg/reflect/#DeepEqual` 进行详细查看：

![reflect.DeepEqual 完整说明](https://image.eddycjy.com/aefb19ae4bce5bbbada39244141bfd68.jpg)

该方法对 Go 语言中的各种类型都进行了兼容处理和判别，由于这不是本文的重点，因此就不进一步展开了。

## 总结

在本文中，我们针对 Go 语言的结构体（struct）是否能够比较进行了具体例子的展开和说明。

其本质上还是对 Go 语言基本数据类型的理解问题，算是变形到结构体中的具体进一步拓展。

不知道你有没有在 Go 结构体吃过什么亏呢，欢迎在下方评论区留言和我们一起交流和讨论。