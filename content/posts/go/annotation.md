---
title: "Go：我有注解，Java：不，你没有！"
date: 2021-12-31T12:55:11+08:00
toc: true
images:
tags: 
  - go
  - 为什么
---

大家好，我是煎鱼。

作为一位 Go 程序员，你会发现身边的同事大多都拥有其他语言的编写经验。那势必就会遇到一点，要把新学到的知识和以前的知识建立连接。

![图来自网络](https://files.mdnice.com/user/3610/050a9802-1ca2-4634-859d-325a09d418c5.png)

特殊在于，Go 有些特性是其他语言有，他没有的。最经典的就是 N 位 Java 同学寻找 Go 语言的注解在哪里，总要解释。

为此，今天煎鱼就带大家了解一下 Go 语言的注解的使用和情况。

## 什么是注解

### 了解历史

注解（Annotation）最早出现自何处，翻了一圈并没有找到。但可以明确，在注解的使用中，Java 注解最为经典，为了便于理解，因此我们基于 Java 做初步的注解理解。

![](https://files.mdnice.com/user/3610/3c9e2434-c5f4-4d7a-bd2d-a8b582b570c8.png)

在 2002 年，JSR-175 提出了 《[A Metadata Facility for the Java Programming Language](https://jcp.org/en/jsr/detail?id=175)》，也就是为 Java 编程语言提供元数据工具。

这就是现在使用最广泛地注解（Annotation）的来源。
示例如下：

```golang
// @annotation1
// @annotation2
func Hello() string {
        return ""
}
```

在格式上均以 “@” 作为注解标识来使用。

### 注解例子

摘抄自 @wikipedia 的一个注解例子：

```java
  //等同于 @Edible(value = true)
  @Edible(true)
  Item item = new Carrot();

  public @interface Edible {
    boolean value() default false;
  }

  @Author(first = "Oompah", last = "Loompah")
  Book book = new Book();

  public @interface Author {
    String first();
    String last();
  }
  
  // 该标注可以在运行时通过反射访问。
  @Retention(RetentionPolicy.RUNTIME) 
  // 该标注只用于类内方法。
  @Target({ElementType.METHOD})
  public @interface Tweezable {
  }

```

在上述例子中，通过注解去做了一系列的定义、声明、赋值等。若是对语言既有注解不熟，或是做的比较复杂的注解，就会有一定的理解成本。

在业内也常常会说，**注解就是 “在源码上进行编码”**，注解的存在，有着明确的优缺点。你觉得呢？

## 注解的作用

在注解的的作用上，分为如下几点：
1. 为编译器提供信息：注释可以被编译器用来检测错误或支持警告。
2. 编译时和部署时处理：软件工具可以处理注释信息以生成代码、XML文件等。
3. 运行时处理：有些注解可以在运行时检查，并用于其他用途。

## Go 注解在哪里

### 现状 

Go 语言本身并没有原生支持强大的注解，仅限于以下两种：
- 编译时生成：go:generate
- 编译时约束：go:build

但这先按不足以作为一个函数注解来使用，也无法形成像 Python 那样的装饰器行为。

### 为什么不支持

Go issues 上有人提过类似的提案：

![](https://files.mdnice.com/user/3610/9695f163-65e8-456a-8dce-bb8551739016.png)

Go Contributor @ianlancetaylor 给出了明确的答复，Go在设计上更倾向于明确的、显式的编程风格。

思考的优缺点如下：
- 优势：不知道 Go 能从添加装饰器中得到什么好处，没能在 issues 上明确论证。
- 缺点：是明确的，会存在意外设置的情况。

因如下原因，没有接受注解：
- 对比现有代码方法，这种装饰器的新的方法没有提供比现有方法更多的优势，大到足矣推翻原有的设计思路。
- 社区内的投票，支持的也很少（基于表情符号的投票），用户反馈不多。

可能有小伙伴会说了，有注解做装饰器了，代码会简洁不少。

对此 Go 团队的态度很明确：

![](https://files.mdnice.com/user/3610/bb357e12-9b15-4729-9381-977d164b6b04.png)

Go 认为**可读性更重要**，如果只是额外多写一点代码，在权衡后，还是可以接受的。

## 用 Go 实现注解

虽然 Go 语言官方没有原生的完整支持，但开源社区中也有小伙伴已经放出了大招，借助各项周边工具和库来实现特定的函数注解功能。


GitHub 项目分别如下：
- [MarcGrol/golangAnnotations](github.com/MarcGrol/golangAnnotations)
- [u2takey/go-annotation](https://github.com/u2takey/go-annotation)

使用示例如下：

```golang
package tourdefrance

//go:generate golangAnnotations -input-dir .

// @RestService( path = "/api/tour" )
type TourService struct{}

type EtappeResult struct{ ... }

// @RestOperation( method = "PUT", path = "/{year}/etappe/{etappeUid}" )
func (ts *TourService) addEtappeResults(c context.Context, year int, etappeUid string, results EtappeResult) error {
	return nil
}
```

对 Go 注解的使用感兴趣的小伙伴可以自行查阅使用手册。

我们更多的关心，Go 原生都没支持，那么开源库都是如何实现的呢？在此我们借助 [MarcGrol/golangAnnotations](github.com/MarcGrol/golangAnnotations) 项目所提供的思路来讲解。

分为三个步骤：
1. 解析代码。
2. 模板处理。
3. 生成代码。

### 解析 AST

首先，我们需要用用 go/ast 标准库获取代码所生成的 AST Tree 中需要的内容和结构。

示例代码如下：

```shell
parsedSources := ParsedSources{
    PackageName: "tourdefrance",
    Structs:     []model.Struct{
        {
      	     DocLines:   []string{"// @RestService( path = "/api/tour" )"},
      	     Name:       "TourService",
      	     Operations: []model.Operation{
                {
              	    DocLines:   []string{"// @RestOperation( method = "PUT", path = "/{year}/etappe/{etappeUid}"},
              	    ...
                },
            },
        },
    },
}
```
我们可以看到，在 AST Tree 中能够获取到在示例代码中所定义的注解内容，我们就可以依据此去做很多奇奇怪怪的事情了。

### 模板生成

紧接着，在知道了注解的输入是什么后，我们只需要根据实际情况，编写对应的模板生成器 code-generator 就可以了。

我们会基于 text/template 标准库来实现，比较经典的像是 [kubernetes/code-generator](https://github.com/kubernetes/code-generator) 是一个可以参考的实现。

代码实现完毕后，将其编译成 go plugin，便于我们在下一步调用就可以了。

### 代码生成

最后，万事俱备只欠东风。差的就是告诉工具，哪些 Go 文件中包含注解，需要我们去生成的。

这时候我们可以使用 `//go:generate` 在 Go 文件声明。就像前面的项目中所说的：

```
//go:generate golangAnnotations -input-dir .
```

声明该 Go 文件需要生成，并调用前面编写好的 golangAnnotations 二进制文件，就可以实现基本的 Go 注解生成了。

## 总结

今天在这篇文章中，我们介绍了注解（Annotation）的历史背景。同时我们针对 Go 语言目前原生的注解支持情况进行了说明。

也面向为什么 Go 没有像 Java 那样支持强大的注解进行了基于 Go 官方团队的原因解释。如果希望在 Go 实现注解的，也提供了相应的开源技术方案。

**你觉得 Go 语言是否需要强大的注解支持**呢，欢迎你在评论区留言和讨论！