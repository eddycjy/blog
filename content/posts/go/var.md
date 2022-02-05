---
title: "为什么 Go 有两种声明变量的方式，有什么区别，哪种好？"
date: 2022-02-05T15:56:48+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

有一个读者刚入门 Go ，提了一个很有意思的问题：Go 有几种种声明变量的方式，作为初学者，到底用哪种，有什么区别，又为什么要有多种声明方式呢？

为此，煎鱼将和大家一起探索这个问题。

## 变量声明

在 Go 中，一共有 2 种变量声明的方式，各有不同的使用场景。

分别是：
- 标准变量声明（Variable declarations）。
- 简短变量声明（Short variable declarations）

### 标准声明

变量声明创建了一个或多个变量，为它们绑定了相应的标识符，并给每个变量一个类型和初始值。

使用语法：

```go
VarDecl     = "var" ( VarSpec | "(" { VarSpec ";" } ")" ) .
VarSpec     = IdentifierList ( Type [ "=" ExpressionList ] | "=" ExpressionList ) .
```

案例代码：

```go
var i int
var U, V, W float64
var k = 0
var x, y float32 = -1, -2
var (
	i       int
	u, v, s = 1.0, 2.0, "脑子进煎鱼了"
)
```

### 简短声明

一个短变量声明。使用语法：

```
ShortVarDecl = IdentifierList ":=" ExpressionList .
```

案例代码：

```go
s := "煎鱼进脑子了"
i, j := 0, 10
f := func() int { return 7 }
ch := make(chan int)
r, w, _ := os.Pipe()
_, y, _ := coord(p) 
```

## 网友疑惑

在我们群里的 Go 读者提了这问题后，我也搜了搜相关资料。发现在 stackoverflow 上也有人提出了类似的疑惑：

![](https://files.mdnice.com/user/3610/ff663b2b-d65a-4969-b5bb-03bc81a247b5.png)

问题是：使用哪一种声明方式，令人困惑。

题主纠结的原因在于：
- 如果一个只是另一个的速记方式，为什么它们的行为会不同？
- Go 的作者出于什么考虑，让两种方式来声明一个变量（为什么不把它们合并成一种方式）？只是为了迷惑我们？
- 有没有其他方面需要我在使用时留心的，以防掉进坑里？

下面我们结合 stackoverflow 的这个提问内容和回答，进一步展开。

思考一下：标准声明和短声明，这两者的区别的在哪那里，又或是凭喜好随意使用？



## 区别在哪

### 代码块的分组声明

使用包含关键字 var 的声明语法时，和其他 package、import、const、type、var 等关键字一样，是可以进行分组的代码块声明的。

例如：

```go
var (
	i       int
	u, v, s = 1.0, 2.0, "脑子进煎鱼了"
)
```

而短声明，是不支持的。

### 变量的初始值指定

使用标准的变量定义时，我们可以只声明，不主动地定义该变量的初始值（缺省会给零值）。

例如：

```go
var (
	i    int
	s    string
)
```

而短声明则不行，必须要在程序中主动地去对变量定义一个值。

例如：

```go
s := "脑子进煎鱼了"
```

此处即使是定义的空字符串，那也属于是用户侧主动定义的，而非缺省的零值。

### 局部变量，区分作用域

在编写程序时，我们经常会有一些局部变量声明，且作用域是有限的。

可以看看自己的代码，这种时候，我们都会采取短声明的方式。

例如：

```go
for idx, value := range array {
    // Do something with index and value
}

if num := runtime.NumCPU(); num > 1 {
    fmt.Println("Multicore CPU, cores:", num)
}
```

短声明在这类场景下有明确的优势，标准的变量声明在这类场景不讨好。

### 重新声明变量

在 Go 语言规范中有明确提到，短变量声明是可以重新声明变量的，这是一个高频重新声明的覆盖动作。

如下：

```go
var name = "煎鱼.txt"

fi, err := os.Stat(name)
if err != nil {
    log.Fatal(err)
}

data, err := ioutil.ReadFile(name)
if err != nil {
    log.Fatal(err)
}
...
```

上述代码中，err 变量就是不断地被反复定义。在 `if err != nil` 猖狂的现在，短变量在此处的优势，简直是大杀器了。

## 总结

相信很多小伙伴初入门时都为此纠结过一下，又或是很多教程压根就没有说清楚两者变量声明的区别。

在今天这篇文章中，我们介绍了 Go 的两种变量声明放水。并且针对短声明存在的场景进行了说明。

主要是：
- 代码块的分组声明。
- 变量的初始值指定。
- 局部变量，区分作用域。
- 重新声明变量。

你觉得变量声明上，还有没有别的优缺点呢，欢迎在评论区交流：）

## 参考

- [GoLang Variable Declaration](https://www.tutorialandexample.com/golang-variable-declaration/)
- [Why there are two ways of declaring variables in Go, what's the difference and which to use?](https://stackoverflow.com/questions/27919359/why-there-are-two-ways-of-declaring-variables-in-go-whats-the-difference-and-w)
- [What is the best practice when declaring variables in go (golang)? E.G. should I use "var x int = 1" or just "x := 1"?](https://www.quora.com/What-is-the-best-practice-when-declaring-variables-in-go-golang-E-G-should-I-use-var-x-int-1-or-just-x-1)