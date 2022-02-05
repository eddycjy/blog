---
title: "详解 Go 空结构体的 3 种使用场景"
date: 2021-12-31T12:54:51+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

在大家初识 Go 语言时，总会拿其他语言的基本特性来类比 Go 语言，说白了就是老知识和新知识产生关联，实现更高的学习效率。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87c66858abf04b848643b6fa03f4bdab~tplv-k3u1fbpfcp-zoom-1.image)

最常见的类比，就是 “Go 语言如何实现面向对象？”，进一步展开就是 Go 语言如何实现面向对象特性中的继承。

这不仅在学习中才用到类比，在业内的 Go 面试中也有非常多的面试官喜欢问：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1cc9c2b2e174fc4a82506b368591a70~tplv-k3u1fbpfcp-zoom-1.image)

来自读者微信群

在今天这篇文章中，煎鱼带大家具体展开了解这块的知识。一起愉快地开始吸鱼之路。

什么是面向对象
-------

在了解 Go 语言是不是面向对象（简称：OOP） 之前，我们必须先知道 OOP 是啥，得先给他 “下定义”。

根据 Wikipedia 的定义，我们梳理出 OOP 的几个基本认知：

*   面向对象编程（OOP）是一种基于 "对象" 概念的编程范式，它可以包含数据和代码：数据以字段的形式存在（通常称为属性或属性），代码以程序的形式存在（通常称为方法）。
    
*   对象自己的程序可以访问并经常修改自己的数据字段。
    
*   对象经常被定义为类的一个实例。
    
*   对象利用属性和方法的私有/受保护/公共可见性，对象的内部状态受到保护，不受外界影响（被封装）。
    

基于这几个基本认知进行一步延伸出，面向对象的三大基本特性：

*   封装。
    
*   继承。
    
*   多态。
    

至此对面向对象的基本概念讲解结束，想更进一步了解的可自行网上冲浪。

Go 是面向对象的语言吗
------------

“Go 语言是否一门面向对象的语言？”，这是一个日经话题。官方 FAQ 给出的答复是：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb8296e58a6d4c7c882984dcceaaf9c0~tplv-k3u1fbpfcp-zoom-1.image)

是的，也不是。原因是：

*   Go 有类型和方法，并且允许面向对象的编程风格，但没有类型层次。
    
*   Go 中的 "接口 "概念提供了一种不同的方法，我们认为这种方法易于使用，而且在某些方面更加通用。还有一些方法可以将类型嵌入到其他类型中，以提供类似的东西，但不等同于子类。
    
*   Go 中的方法比 C++ 或 Java 中的方法更通用：它们可以为任何类型的数据定义，甚至是内置类型，如普通的、"未装箱的 "整数。它们并不局限于结构（类）。
    
*   Go 由于缺乏类型层次，Go 中的 "对象 "比 C++ 或 Java 等语言更轻巧。
    

Go 实现面向对象编程
-----------

### 封装

面向对象中的 “封装” 指的是可以隐藏对象的内部属性和实现细节，仅对外提供公开接口调用，这样子用户就不需要关注你内部是怎么实现的。

在 Go 语言中的属性访问权限，通过首字母大小写来控制：

*   首字母大写，代表是公共的、可被外部访问的。
    
*   首字母小写，代表是私有的，不可以被外部访问。
    

Go 语言的例子如下：

```
type Animal struct {
 name string
}

func NewAnimal() *Animal {
 return &Animal{}
}

func (p *Animal) SetName(name string) {
 p.name = name
}

func (p *Animal) GetName() string {
 return p.name
}

```

在上述例子中，我们声明了一个结构体 `Animal`，其属性 `name` 为小写。没法通过外部方法，在配套上存在 Setter 和 Getter 的方法，用于统一的访问和设置控制。

以此实现在 Go 语言中的基本封装。

### 继承

面向对象中的 “继承” 指的是子类继承父类的特征和行为，使得子类对象（实例）具有父类的实例域和方法，或子类从父类继承方法，使得子类具有父类相同的行为。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec1cf116a1bf404981cb624009945157~tplv-k3u1fbpfcp-zoom-1.image)

图来自网络

从实际的例子来看，就是动物是一个大父类，下面又能细分为 “食草动物”、“食肉动物”，这两者会包含 “动物” 这个父类的基本定义。

在 Go 语言中，是没有类似 `extends` 关键字的这种继承的方式，在语言设计上采取的是组合的方式：

```
type Animal struct {
 Name string
}

type Cat struct {
 Animal
 FeatureA string
}

type Dog struct {
 Animal
 FeatureB string
}

```

在上述例子中，我们声明了 `Cat` 和 `Dog` 结构体，其在内部匿名组合了 `Animal` 结构体。因此 `Cat` 和 `Dog` 的实例都可以调用 `Animal` 结构体的方法：

```
func main() {
 p := NewAnimal()
 p.SetName("煎鱼，记得点赞~")

 dog := Dog{Animal: *p}
 fmt.Println(dog.GetName())
}

```

同时 `Cat` 和 `Dog` 的实例可以拥有自己的方法：

```
func (dog *Dog) HelloWorld() {
 fmt.Println("脑子进煎鱼了")
}

func (cat *Cat) HelloWorld() {
 fmt.Println("煎鱼进脑子了")
}

```

上述例子能够正常包含调用 `Animal` 的相关属性和方法，也能够拥有自己的独立属性和方法，在 Go 语言中达到了类似继承的效果。

### 多态

面向对象中的 “多态” 指的同一个行为具有多种不同表现形式或形态的能力，具体是指一个类实例（对象）的相同方法在不同情形有不同表现形式。

多态也使得不同内部结构的对象可以共享相同的外部接口，也就是都是一套外部模板，内部实际是什么，只要符合规格就可以。

在 Go 语言中，多态是通过接口来实现的：

```
type AnimalSounder interface {
 MakeDNA()
}

func MakeSomeDNA(animalSounder AnimalSounder) {
 animalSounder.MakeDNA()
}

```

在上述例子中，我们声明了一个接口类型 `AnimalSounder`，配套一个 `MakeSomeDNA` 方法，其接受 `AnimalSounder` 接口类型作为入参。

因此在 Go 语言中。只要配套的 `Cat` 和 `Dog` 的实例也实现了 `MakeSomeDNA` 方法，那么我们就可以认为他是 `AnimalSounder` 接口类型：

```
type AnimalSounder interface {
 MakeDNA()
}

func MakeSomeDNA(animalSounder AnimalSounder) {
 animalSounder.MakeDNA()
}

func (c *Cat) MakeDNA() {
 fmt.Println("煎鱼是煎鱼")
}

func (c *Dog) MakeDNA() {
 fmt.Println("煎鱼其实不是煎鱼")
}

func main() {
 MakeSomeDNA(&Cat{})
 MakeSomeDNA(&Dog{})
}

```

当 `Cat` 和 `Dog` 的实例实现了 `AnimalSounder` 接口类型的约束后，就意味着满足了条件，他们在 Go 语言中就是一个东西。能够作为入参传入 `MakeSomeDNA` 方法中，再根据不同的实例实现多态行为。

总结
--

通过今天这篇文章，我们基本了解了面向对象的定义和 Go 官方对面向对象这一件事的看法，同时针对面向对象的三大特性：“封装、继承、多态” 在 Go 语言中的实现方法就进行了一一讲解。

在日常工作中，基本了解这些概念就可以了。若是面试，可以针对三大特性：“封装、继承、多态” 和 五大原则 “单一职责原则（SRP）、开放封闭原则（OCP）、里氏替换原则（LSP）、依赖倒置原则（DIP）、接口隔离原则（ISP）” 进行深入理解和说明。

在说明后针对上述提到的概念。再在 Go 语言中讲解其具体的实现和利用到的基本原理，互相结合讲解，就能得到一个不错的效果了。

## 鼓励

若有任何疑问欢迎评论区反馈和交流，**最好的关系是互相成就**，各位的**点赞**就是[煎鱼](https://github.com/eddycjy)创作的最大动力，感谢支持。

> 文章持续更新，可以微信搜【脑子进煎鱼了】阅读，本文 **GitHub** [github.com/eddycjy/blog](https://github.com/eddycjy/blog) 已收录，欢迎 Star 催更。


参考
--

*   Is Go an Object Oriented language?
    
*   面向对象的三大基本特征，五大基本原则
    
*   Go 面向对象编程（译）