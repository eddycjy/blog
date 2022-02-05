---
title: "Go 为什么不支持类和继承？"
date: 2021-12-31T12:55:22+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

大家在早期学习 Go 时，一旦跨过语法的阶段后。马上就会进入到一个新的纠结点，Go 不支持面向对象吗？

![](https://files.mdnice.com/user/3610/a299d98d-e46c-4a6d-8362-02f957e86b10.png)


这门编程语言里没有类（class）、继承（extends），~~没法一把搜了，面试问啥面向对象（OOP）~~？

今天煎鱼就带大家一起来了解这之中的思考，Go 真的不支持吗？

## 类和继承

### 类是什么

类（class）在面向对象编程中是一种面向对象计算机编程语言的构造，是创建对象的蓝图，描述了所创建的对象共同的特性和方法（via @维基百科）。

例子如下：

```php
class SimpleClass
{
    // 声明属性
    public $var = '脑子进煎鱼了';

    // 声明方法
    public function displayVar() {
        echo $this->var;
    }
}
```

每个类的定义都以关键字 class 开头，后面跟着类名，后面跟着一对花括号，里面包含有类的属性与方法的定义。

### 继承是什么

继承是面向对象软件技术当中的一个概念，如果一个类别 B “继承自”另一个类别 A，就把这个 B 称为 “A的子类”，而把 A 称为 “B的父类别” 也可以称 “A 是 B 的超类”（via @维基百科）。

例子如下：

```php
// 父类
class Foo
{
    public function printItem($string)
    {
        echo '煎鱼1: ' . $string . PHP_EOL;
    }
    
    public function printPHP()
    {
        echo 'PHP is great.' . PHP_EOL;
    }
}

// 子类
class Bar extends Foo
{
    public function printItem($string)
    {
        echo '煎鱼2: ' . $string . PHP_EOL;
    }
}
```

继承有如下两个特性：
- 子类具有父类别的各种属性和方法，不需要再次编写相同的代码。
- 子类别继承父类时，可以重新定义某些属性，并重写某些方法，使其获得与父类别不同的功能。

## 结构和组合

在 Go 里就比较 ”特别“ 了，因为没有传统的类，也没有继承。

取而代之的是结构和组合的方式。这也是业内对 Go 是否 OOP 争议最大的地方。

### 结构体

我们可以在 Go 中通过结构体的方式来组织代码，达到类似类的方式。

例子如下：

```golang
package main

import "fmt"

type person struct {
    name string
    age  int
}

func(p *person) hello(){}

func newPerson(name string) *person {
    p := person{name: name}
    p.age = 42
    return &p
}

func main() {
    fmt.Println(person{"煎鱼1", 22})
    fmt.Println(person{name: "煎鱼2", age: 33})
    ...
}
```

在上述代码中，我们可以定义结构体内的属性，也可以针对结构体这些类型定义只属于他们的方法。

在声明实例上，可以配合 `newXXX` 的初始化方法来生成，这是 Go 里约定俗成的方式。

### 组合

类的声明采取结构体的方式取代后，也可以配套使用 ”组合“ 来达到类似继承的效果。

例子如下：

```golang
type man struct {
	name string
}

func (m *man) hello1() {}

type person struct {
	man
	name string
}

func (p *person) hello2() {}

func newPerson(name string) *person {
	p := person{name: name}
	return &p
}

func main() {
	p := newPerson("脑子进煎鱼了")
	p.hello1()
}
```

在上述代码中，我们分别定义了 man 和 person 两个结构体，并将 man 嵌入到 person 中，形成组合。

你可以在 main 方法中能够看到，person 实例是可以使用和调用 man 实例的一些公开属性和方法的。

在简单的使用效果上会与继承有些接近。

## Go 是面向对象的语言吗

“Go 语言是否一门面向对象的语言？”，这是一个日经话题。官方 FAQ 给出的答复是：

![](https://files.mdnice.com/user/3610/159601a9-b428-4958-9f85-2214aea30127.png)

是的，也不是。原因是：

- Go 有类型和方法，并且允许面向对象的编程风格，但没有类型层次。
- Go 中的 "接口 "概念提供了一种不同的方法，我们认为这种方法易于使用，而且在某些方面更加通用。还有一些方法可以将类型嵌入到其他类型中，以提供类似的东西，但不等同于子类。
- Go 中的方法比 C++ 或 Java 中的方法更通用：它们可以为任何类型的数据定义，甚至是内置类型，如普通的、"未装箱的 "整数。它们并不局限于结构（类）。
- Go 由于缺乏类型层次，Go 中的 "对象 "比 C++ 或 Java 等语言更轻巧。

## 为什么不支持类和继承

有的人认为类和继承是面向对象的必要特性，必须要有，才能是面向对象的语言，但其实也并非如此。

面向对象（OOP）有不同的含义和解读，许多概念也可以通过结构体、组合和接口等方式进行表达，说是不支持传统的 OOP。

其实真相是 Go 是选择了另外一条路，也就是 ”**组合优于继承**“。我们所提到的类和继承并不是定义 OOP 的一种准则，只是协助完成 OOP 的方法之一。

不要本末倒置了，不让工具来定义 OOP 的理念。

## 总结

在今天这篇文章中，我们介绍了常说的类和继承的业内定义和使用案例。同时面向 Go 读者群里的疑惑，进行了解答。

实质上，Go 是 OOP，也不是 OOP。类和继承只是实现 OOP 的一种方式，但并不是没有这两者，他就不是 OOP 了。

不支持的原因也很明确，Go 在设计上，选择了组合优于继承的编程设计模式，它不是传统那种面向类型的范式。

你觉得呢，欢迎大家在评论区留言和交流：）
 
## 参考

- 为什么人们声称 Go 不是面向对象的，我是不是误读了什么？：https://www.reddit.com/r/golang/comments/a9rn6n/why_people_claim_that_go_is_not_object_oriented/
- 组合大于继承：https://en.wikipedia.org/wiki/Composition_over_inheritance
- Go 是面向对象的吗？：https://flaviocopes.com/golang-is-go-object-oriented/
- 类 vs 新型 + 接收器？：https://www.reddit.com/r/golang/comments/a8zgvm/class_vs_new_type_receiver/
- 结构而不是类：https://golangbot.com/structs-instead-of-classes/