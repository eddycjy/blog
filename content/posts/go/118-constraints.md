---
title: "Go 新语法挺丑？最新的泛型类型约束介绍"
date: 2021-12-31T12:55:19+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

近期我们在分享《[3 件与 Go 开发者有关的大小事](https://mp.weixin.qq.com/s/22TeQOUjf_glPX3QLPX8yw)》时，里面有一部分有提到 Go 泛型的约束类型语法又又又变了。

在评论区里看到不少的读者朋友大呼泛型的新类型约束语法挺丑，不如原本的好...

如下：

![](https://files.mdnice.com/user/3610/8e734a8f-612c-41c3-a0cd-8a5fe6eea483.png)

为此，今天煎鱼就带大家来看看，为什么会发生泛型的新语法会这种改变？

## 问题背景

原本 @Ian Lance Taylor 设计的的泛型类型关键字如下：

```golang
type T interface {
 type int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64, uintptr, float32, float64, complex64, complex128, string
}
```
看起来好像非常 “顺眼”。但在《[proposal: Go 2: sum types using interface type lists](https://github.com/golang/go/issues/41716 "proposal: Go 2: sum types using interface type lists")》中社区进行了热烈的讨论。

认为该类型约束的关键字，过于 “模棱两可”。像是 @Damien Neil 所提出的以下两个例子。

结构体的例子：

```golang
package p
type mustIncludeDefaultCase struct{}
type MySum interface {
  type int, float64, mustIncludeDefaultCase
}
```

不明确的点之一，如果类型列表包含一个未导出的类型，那又应该是如何处理呢？

接口的例子：

```golang
type T interface { type int16, int32 }
func main() {
  var x T
  switch x.(type) {
  case int16:
  case int32:
  } 
}
```

你认为程序会跑进哪个 switch-case 的代码块里呢，是 int16，还是 int32？

不，都不会，变量 x 是 nil，如此迷惑。

在社区讨论中，发现设计与真实场景一结合，发现这个类型规则在普通的接口类型、在约束中使用也太微妙了。

用类型列表嵌入接口时的行为也很奇怪。认为可以做的更好，那就是 “更显式”。

## 新提案

为此，Go 泛型的设计者 @Ian Lance Taylor 提出了一个新的提案《[spec: generics: use type sets to remove type keyword in constraints](https://github.com/golang/go/issues/45346 "spec: generics: use type sets to remove type keyword in constraints")》。

其包含三个新的、更简单的想法来取代泛型提案中定义的类型列表。

### 关键名词

新语法在泛型处增加一个新概念：接口元素（interface elements），用作约束条件的接口类型，或者被嵌入约束条件的接口类型，允许嵌入一些额外的构造。

被嵌入的可以是：
- 任何类型，而不仅仅是一个接口类型。
- 一个新的句法结构，称为近似元素。
- 一个新的句法结构，称为联合元素。

重点名词，我们继续展开讲解，分别是：
- 嵌入约束。
- 近似元素。
- 联合元素。
- 接口类型。

### 联合元素

原先的语法中，类型约束会用逗号分隔的方式来展示。

如下：
```golang
type int, int8, int16, int32, int64
``` 

在新语法中，结合定义为 union element（联合元素），写成一系列由竖线 ”|“ 分隔的类型或近似元素。

如下：

```golang
type PredeclaredSignedInteger interface {
	int | int8 | int16 | int32 | int64
}
```

常常会和下面讲到的近似元素一起使用。

### 近似元素

新语法，他的标识符是 “~”，完整用法是 `~T`。`~T` 是指底层类型为 T 的所有类型的集合。例如：

```golang
type AnyInt interface{ ~int }
```

他的类型集是 `~int`，也就是所有类型为 int 的类型（如：int、int8、int16、int32、int64）都能够满足这个类型约束的条件。

再结合以上的分隔来使用，用法为：

```golang
type SignedInteger interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}
```

相当于泛型提案中使用的以下类型：

```golang
interface {
	type int, int8, int16, int32, int64
}
```

新语法只需借助近似标识符 `~int` 来表达就可以了，更明确的表示了近似匹配，而不是存在隐式理解。

### 嵌入约束

一个类型约束可以嵌入另一个约束，联合元素可以包括约束。

例如：

```golang
// Signed is a constraint whose type set is any signed integer type.
type Signed interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}

// Unsigned is a constraint whose type set is any unsigned integer type.
type Unsigned interface {
	~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

// Float is a constraint whose type set is any floating point type.
type Float interface {
	~float32 | ~float64
}

// Ordered is a constraint whose type set is any ordered type.
// That is, any type that supports the < operator.
type Ordered interface {
	Signed | Unsigned | Float | ~string
}
```

这个很好理解，就是正式支持嵌入约束了。

### 接口类型（联合约束元素）

在联合元素中，使用接口类型的话。将会把该类型集添加到联合中。

例如：

```golang
type Stringish interface {
	string | fmt.Stringer
}
```

Stringish 的类型集将是字符串类型和所有实现 `fmt.Stringer` 的类型，任何这些类型（包括 `fmt.Stringer` 本身）将被允许作为这个约束的类型参数。
 
也就是针对接口类型做了特殊的处理。

## 总结

今天这篇文章，主要讲解了 Go 泛型的新语法，其实本质上还是为了解决引入 A 后，出现了 BCD 新问题，又继续引入新的语法和模式来解决。

整体就是三个观点：

- 显式匹配：使用**明确的 "~" 近似元素，澄清了何时在底层类型上进行匹配**。
- 明确含义：**使用 “|” 而不是 “,” 强调这是一个元素的联合**。
- 嵌套优化：通过允许约束嵌入非界面元素，类型关键字可以被省略。

这就是用这两个新语法符号的原因，被嫌丑的新语法标识符 “|” 和 “~” ，其实也是在 issues 的大范围讨论中，由社区贡献出来的。

算是有利有弊？**你的看法如何，欢迎在评论区留言**：）
