---
title: "长达 12 年，Go 才引入泛型，是政治，还是技术问题？"
date: 2021-12-31T12:55:25+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

前两天 Go1.18 beta1 已经发布，距离正式发布 Go1.18 的生产可用还有 2 个月，也就是泛型即将正式面世。

最近正在收集泛型的一些资料，看到在 2015 年有人在 Hacker News 上的《[Go 1.5 max procs default](https://news.ycombinator.com/item?id=9622417 "Go 1.5 max procs default")》吐槽 Go 不支持泛型是 “政治” 原因...

看了还是有些意义的，**与现在的矛盾点基本一致**，为此分享给大家。

## 网友吐槽

网友 @aikah 认为 Go 团队不太可能在语言中加入泛型，这显然是一个政治问题而不是技术问题。错误处理也是如此。

和许多人一样，该网友**认为 Go 在极简主义和功能之间没有取得正确的平衡**。反对泛型的人赞成用编译时类型检查（总是安全的）换取运行时类型断言（可能失败）。

他们拒绝承认这一事实。这就是他们反对泛型的论点，并将最终损害语言的任何潜在增长。他们基本上是在违背自己的利益。

## 官方回复

Russ Cox 做了正式的回复：很抱歉，但不是：**泛型是一个技术问题**，不是一个政治问题。

Go 团队并**不反对泛型本身**，只是反对做那些没有被很好理解或不能很好地与 Go 配合的事情。

这就是核心观点和矛盾点，也从 2009 年，延续到了现在。

### 会遇到的问题

Go 团队认为要将泛型的概念融入 Go，并与系统的其他部分很好地配合，必须解决一些深层次的技术问题，而我们并没有解决这些问题的办法。

关于这些问题，在几年前就在博客上写过一篇《[The Generic Dilemma](https://research.swtch.com/generic "The Generic Dilemma")》：

![](https://files.mdnice.com/user/3610/8ed3923a-d5fc-482f-bb6d-6fff7a19fcaa.png)

即使克服了那一页上的问题，也有其他问题，接下来你会遇到的问题是：”如何让程序员以一种有用的、易于解释的方式省略类型注释“。

也就是如何更人性、更易于的表达泛型的类型参数。

#### 泛型例子

举个例子，C++ 允许你写 `make_pair(1, "foo")`，而不是 `make_pair<int, string>(1, "foo")`。

为了达到这种效果，推断注释背后的逻辑需要几页几页的规范，这并不是一个特别容易理解的编程模型，当事情出错时，编译器也不能轻易解释。

在这之后肯定还有更多的新问题在这里面。

#### 和专家沟通

Go 团队和一些真正的 Java 泛型专家谈过，他们每个人都说了大致相同的话：要非常小心，它不像看起来那么容易，而且你会被你犯的所有错误困住。

作为一个 Java 示范，可以浏览一下《[Java Generics FAQs - Frequently Asked Questions](http://www.angelikalanger.com/GenericsFAQ/JavaGenericsFAQ.html "Java Generics FAQs - Frequently Asked Questions")》的大部分内容：

![](https://files.mdnice.com/user/3610/6e188c16-f31c-4d6b-9d1b-216aa702d548.png)

看看过了多久你会开始思考 "这真的是最好的方法吗？"。

在泛型过程中会遇到许多问题，像是《[How do I decrypt Enum<E extends Enum<E>>](http://www.angelikalanger.com/GenericsFAQ/FAQSections/TypeParameters.html#FAQ106" "How do I decrypt Enum<E extends Enum<E>>")》：

![](https://files.mdnice.com/user/3610/2cbd8083-cee6-48f7-a0bd-d6faf8d27c40.png)

为此，Go 团队在泛型的推动上非常谨慎。
  
## 承认缺点

Go 团队说得很清楚，承认这个事实：**没有泛型是有一定的缺点的**。
  
你要么使用 `interface{}` 而放弃编译时检查，要么写代码生成器而使你的构建过程复杂化。

现有语言中实现的泛型也有明确的缺点，而且今天不妥协有一个非常大的好处：它使明天**采用更好的解决方案变得更加容易**。

## 总结
  
今天给大家分享了过去在国外社区针对 Go 泛型的各种争议和探讨，其实泛型的核心观点很明确：”Go 团队不反对泛型本身“。
  
一直没能把泛型做起来，也是因为顾虑很多，做泛型要和 Go 多个部分，要解决很多深层次的问题，还要解决类型参数可读性等问题。所以一直拖到了现在。
  
回到即将 2022 年的现在，都预言对了。社区都在扯做泛型后的多个关联组件，以及泛型可读性和结构...

显然，**泛型就是双刃剑？你怎么看**。