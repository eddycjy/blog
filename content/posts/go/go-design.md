---
title: "上帝视角：Go 语言设计失误，缺乏远见？"
date: 2021-12-31T12:55:13+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

前段时间我有一个朋友在某乎上摸鱼时，给我甩来一个主题为《golang 设计者是如何偿还技术债的》链接。

说是让我学习、围观一下社区观点，早日好修成正果，本鱼表示满脸问号。

原回答如下图：

![](https://files.mdnice.com/user/3610/4bbdfe91-b98e-4ae1-bcc6-aa5fc50cd184.png)

主要是以极短的话语表述 Go 语言的 “泛型、异常、channel、annotation、模块依赖” 的设计是失误的。

说是没有向各种编程语言的 “最佳实践” 各取所需。

## 那些故事

刚好煎鱼也入门 Go 没几天，偶尔翻过 issues 和 proposal，看了一点点历史事件。

![图来自 Introduction to Golang](https://files.mdnice.com/user/3610/3213b564-1a53-4dcf-9b69-4f4359bd50db.png)

也从我的观点来围观一下 Go 官方这些年为特性挣扎过的那些事。

涉及：
1. 泛型。
2. 错误处理。
3. 依赖管理。
4. 注解。

### 泛型

**为什么 Go 语言这么久都没有泛型**，是不是 Go 官方不够 “聪明”，抄作业都不会抄。这显然是不对的。

有如下几点原因：
1. 泛型本质上并不是绝对的必需品。
2. 泛型不是 Go 语言的早期目标。
3. 其他 feature 更重要，把精力放在这些上面，Go 团队人力很有限的。

#### 历史尝试

在以往的尝试中，Go 团队有人进行过不少的泛型 proposal 试验。基本时间线（via @changkun）如下：

|  简述   | 时间  | 作者 |
|  ----  | ----  | --- |
| Type Functions  | 2010年 | Ian Lance Taylor |
| Generalized Types  | 2011年 | Ian Lance Taylor |
| Generalized Types v2  | 2013年 | Ian Lance Taylor |
| Type Parameters  | 2013年 | Ian Lance Taylor |
| go:generate  | 2014年 | Rob Pike |
| First Class Types  | 2015年 | Bryan C.Mills |
| Contracts  | 2018年 | Ian Lance Taylor, Robert Griesemer |
| Contracts  | 2019年 | Ian Lance Taylor, Robert Griesemer  |
| Redundancy in Contracts(2019)'s Design  | 2019年 | Ian Lance Taylor, Robert Griesemer |
| Constrained Type Parameters(2020, v1)  | 2020年 | Ian Lance Taylor, Robert Griesemer |
| Constrained Type Parameters(2020, v2)  | 2020年 | Ian Lance Taylor, Robert Griesemer |
| Constrained Type Parameters(2020, v3)  | 2020年 | Ian Lance Taylor, Robert Griesemer |

我们观察一下，10 年过去了，Ian Lance Taylor 依然在开展泛型提案，持续地在思考着 Go 泛型。

坚持思考，这一点值得我们学习。

对 Go 泛型历史有兴趣的读者可以看看《[为什么 Go 的泛型一拖再拖？](https://mp.weixin.qq.com/s/ftmuA9g7QPAwSwiswRuSuw)》，给了明确完整的内容介绍和过程描述了。

### 下一步计划

在 2021 年尾巴的我们，明年（2022年） **Go1.18 左右就可以见到 Go 泛型，基本跑不了。**

想想就激动，如下图（此刻是 4 个月后）：

![图来自网上](https://files.mdnice.com/user/3610/934613fa-2564-4f26-bd40-bb309bd48dcf.png)

在出来前可以看看《[Go 1.17 支持泛型了？具体怎么用](https://mp.weixin.qq.com/s/Pf7YuFpwbldSB60DDCBtlA)》，可以作为玩具用了。

接下来可以预见泛型出来后，一堆工具库和数据结构很大可能会被逐步改写，像是《[Go 提案：增加泛型版 slices 和 maps 新包](https://mp.weixin.qq.com/s/D7u7nxixctoFIL-Pch0zvw)》，早已摩拳擦掌。

届时 Go 源码类别的书的部分内容也会失时效，需要关注 Go 版本的时效性。

### 错误处理

在日常工程中，我们写的、看到最多的可能就是这一段标志性 Go 代码：

```golang
func main() {
 x, err := foo()
 if err != nil {
   // handle error
 }
 y, err := foo()
 if err != nil {
   // handle error
 }
 z, err := foo()
 if err != nil {
   // handle error
 }
 s, err := foo()
 if err != nil {
   // handle error
 }
}
```

这是在业内被吐槽的最多的，甚至都可以用来作为 Gopher 的互认。

#### 设计方向

那 Go 是瞎设计的吗，就粗制滥造，搞个错误 err 的返回约定惯例。像是：

```golang
func foo() err {
    return nil
}
```

其实并不是，Go 团队在设计上有意识地选择了**显式**的设计方向，如下：
- 使用显式错误结果。
- 使用显式错误检查。

这和其他语言不一样 ，是由于 Go 团队也认识到了异常处理的不可见错误检查所带来的问题。

设计草案有一部分是受到了这些问题的启发。如下：

![](https://files.mdnice.com/user/3610/9c308232-631e-4aa3-b265-85daf2b9909d.png)

目前 Go 官方也没有打算去掉 “显式” 这一做法，新版 Go2 错误处理的核心目标是：“**错误检查更加轻便，减少专门用于错误检查的 Go 程序代码的数量和所花费的时间**。”。

从 Go2 的趋势来看，主要是增加关键字和修饰来解决这个问题，相当于是堆积木了，而不是直接把他干掉的。

这在 Go 核心团队内是非常明确的。

#### 进一步深入

对 Go 语言错误处理还想进一步深入的，推荐看下面这几篇文章：

- 《[先睹为快，Go2 Error 的挣扎之路](https://mp.weixin.qq.com/s/XILveKzh07BOQnqxYDKQsA)》
- 《[Go errors 不会有进一步的改进计划](https://mp.weixin.qq.com/s/ixBMcAgqW51I0r_hkw5l5A)》
- 《[你对 Go 错误处理的 4 个误解！](https://mp.weixin.qq.com/s/Ey-yqIq__wpaLTlBAOHjxg)》

### 依赖管理

Go 语言在一开始是完全基于 GOPATH 作为依赖管理的模式，当时也闹了不少的争议出来。有以下核心问题：
1. 依赖要手动拉取和下载，没有强版本化的概念，开发者很难受（例如：不兼容升级、要拉取同一份）。
2. 依赖和工程代码必须在 GOPATH 下才能运行，不能任意摆放。

所以在 Go1.0~Go1.11 中，各路神仙发招，社区出现了各种诸如 dep、glide、godep 等依赖包管理工具。

#### 时间线

后续 Go 团队在 Russ Cox 的强势推进下，力排众议，推动 Go modules 的发展：

![](https://files.mdnice.com/user/3610/66291d6e-b10b-42db-9242-d5e16eb94237.png)

时间线如下：
- Go1.11 起开始推进 Go modules（前身 vgo）。
- Go1.13 起不再推荐使用 GOPATH 的使用模式。
- Go1.14 表示已经准备好，可以用在生产上（ready for production）了。

#### 为什么这么晚

为什么 Go modules 这么晚才诞生，这是不是就是 Go 团队的设计失误呢？

我认为，是也不是。

Go 的诞生一开始是为了解决 Google 几位大佬自己的痛点。

在 Google 的依赖管理上，本身是大仓库（Monorepo）的模式，企业内部有自己一整套工具和流程，设计之初没有这块的强诉求。

如下：

![图来自 Mono Repo vs Multi Repo](https://files.mdnice.com/user/3610/c589a0af-e0c1-4e9b-83c2-899369c9fa8e.jpg)

有兴趣的读者详细可阅读《[Why Google Stores Billions of Lines of Code in a Single Repository](https://dl.acm.org/doi/epdf/10.1145/2854146)》，

Go 在社区开源后，大规模使用后这个问题就爆发了，社区自行释出了方案。可惜，五花八门，也都没有解决好。官方队伍就自己上手了。

要知道，没有技术方案是完美的。Go modules 也被不少人所吐槽，存在争议。

#### 进一步深入

想更进一步深入 Go modules 的小伙伴，可以看看下述文章：
- 《[Go Modules 终极入门和历史](https://mp.weixin.qq.com/s/6gJkSyGAFR0v6kow2uVklA)》
- 《[干货满满的 Go Modules 和 goproxy.cn](https://mp.weixin.qq.com/s/jpp7vs3Fdg4m15P1SHt1yA)》

### 注解

Go 开发者中有大部分同学都有其他语言的使用经验。在其他语言中，注解是一个强大的工具，没得用会很不习惯。

![图片来自网络](https://files.mdnice.com/user/3610/9061ff09-9776-4a74-9550-fc5a4f17f824.png)

甚至有听过没有注解，就自嘲不会 “写” 代码了，所以一上来就找 Go 语言的注解怎么用了。

#### 一些疑惑

我有一个朋友，经常会听到如下疑惑，甚至无奈的发问：

- “怎么样在函数前声明，直接开启事务？”
- "为什么 Java 可以完美注解，Go 就不行，难以理解，我无法接受..."
- “那 Go 支持什么程度的注解？”

Go 的 “注解” 支撑的非常有限，基本都是 `//go build`、`go:generate` 这类辅助，达不到标准的装饰器的作用。

#### 为什么不支持

没有全面的支持注解来做装饰器，显然不算 Go 的设计失误，这是刻意为之，这是与错误处理的设计理念相关联。

Go issues 上有人提过类似的提案：

![](https://files.mdnice.com/user/3610/9695f163-65e8-456a-8dce-bb8551739016.png)

Go Contributor @ianlancetaylor 给出了明确的答复，Go在设计上更倾向于明确的、显式的编程风格。

优缺点如下：
- 优势：不知道 Go 能从添加装饰器中得到什么好处，没能在 issues 上明确论证。
- 缺点：是明确的，会存在意外设置的情况。

因如下原因，没有接受注解：
- 对比现有代码方法，这种装饰器的新的方法没有提供比现有方法更多的优势，大到足矣推翻原有的设计思路。
- 社区内的投票，支持的也很少（基于表情符号的投票），用户反馈不多。

可能有小伙伴会说了，有注解做装饰器了，代码会简洁不少。

但其实 Go 团队的态度很明确：

![](https://files.mdnice.com/user/3610/bb357e12-9b15-4729-9381-977d164b6b04.png)

Go 认为**可读性更重要**，如果只是额外多写一点代码，在权衡后，还是可以接受的。

如果想自己在 Go 中实现完整注解的，可以详细阅读《[Go：我有注解，Java：不，你没有！](https://mp.weixin.qq.com/s/hrsagmDtjt6r9fJKf8SUcQ)》，可以给到你一些思路。

## 偿还的过程

如果是在职场中工作多年的小伙伴，其实不难发现 Go 的发展史和业务的发展节奏是类似的。

在社区中吐槽的主要是两块，如下：
- 为什么这个功能不如此设计？
- 这个功能为什么没有支持？

### 不如此设计

为什么 Go 语言不如此设计？经典的像是 Go 的错误处理（error），很多小伙伴会**先入为主**，以其他语言的最佳实践，要教 Go 团队设计，要 throw，要 catch！

其实想一下，我们做一个业务，这个业务就是 Go 语言。我们需要先做业务建模，确定 Go 的核心思想，才能持续的迭代和设计。

Go 语言的设计定义很明显是：**既要简单、还要显式，不能有隐式、要避免复杂**，所以社区传递的是 “**less is more**” 的设计理念。

这么想，很多提案的落地，被拒等，都能了解到 Go 语言的设计哲学和团队理念。

### 还没有支持

为什么 Go 语言的 XXX 功能没有支持？经典的像是 Go 的泛型、注解等功能。

还没有支持的可能性有三点，如下：
1. 还没有想清楚。
2. 早就被拒绝了。
3. 优先级不够高。

实际上和我们业务迭代一样，Go 团队的人力资源有限，**做事会有优先级**。前文所提到的 Russ Cox 就是现在 Go 团队 Leader，每年也会开相关的会议讨论事项。

像是 Go 泛型，显然没有，也不会影响到 Go 在业务初期的短期发展，国内依然存有一定的占用率。2011 年没有想清楚，也就一直持续思考和尝试了...

而注解，或是你们想到的。很多在 go issues 其实早就被拒绝过多次，自然还没有支持，也是因为他不大可能直接出现了。

### 推进的模式

Go 在推进或偿还新技术改进时，现在采取的模式都是一样的。会先设计一个编译时可以指定的 “变量”。

例如：
- 泛型的 G 变量。
- Modules 的 GO111MODULE 变量。

再在 Go 的不断迭代中，推进使用和反馈，再推进变量的默认开启，逐渐去除。

可以参考 GO111MODULE 的过程。

## 总结

我们在学习很多语言、技能时，会以既有的知识去认知，再对新的对象建立新的认知树，很容易会有**先入为主的认知行为**。

但若没有及时思考，就很容易产生偏见。**认为 XXX 是 XXX，你 Go 语言就应该是 XXX，这样是有失偏颇的**。

就像我们行业经常讨论的，网上的 A 同学，35 岁被裁员了。那你我，35 岁就 100% 会下岗吗？

相反，Go 语言这 10+ 年来，基于自己的设计理念。保持了大致一贯的 less is more 设计理念，是值得赞许的。

我们要知道软件设计，是**没有银弹**的。Go 语言的设计理念，有好有坏，**社区也有不少人对大道至简的理念嗤之以鼻**。

**你又是怎么看待 Go 语言的呢**，欢迎点赞、留言，一起来交流和讨论：）