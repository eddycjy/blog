---
title: "Rust 内讧，PHP 主力淡出？Go：好好放假"
date: 2021-12-31T12:55:20+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

现在已经是 2021 年的 Q4 季度了，许多职场人都忙的飞起，被 PPT 各种轰炸。

在上周，看到几门语言的社区都发生了一些大事，煎鱼表示大受震撼，来说几句我的看法。

## PHP 主力淡出

在 11 月 23 日，看到 PHP 的主力开发 [Nikita Popov](https://twitter.com/nikita_ppv "Nikita Popov") 在论坛上发文宣布将**不再以专业身份从事 PHP 工作**，投入到 PHP 开发中的时间将会大幅度减少。

![](https://files.mdnice.com/user/3610/82ba0eaf-fa69-4bee-9cbc-ec385d3b9793.png)

根据 Jetbrains 分享的消息来看，可得知 Nikita Popov 在高中（2011 年）时就开始参与 PHP 开发，截止现在已有 10 年经验了。

![](https://files.mdnice.com/user/3610/93d9d4c9-9863-42fe-9d2c-f5ada62210c2.png)

他离开的原因，我看了一遍帖子，众说纷纭，业内猜测有以下两个观点：
- 迫于生活压力，过多精力投入维护开源项目收入不高。
- PHP 新版本特性受阻等原因，把精力从 PHP 转到 LLVM。

也因此，PHP 社区加速宣布成立 PHP 基金会《[The New Life of PHP – The PHP Foundation](https://blog.jetbrains.com/phpstorm/2021/11/the-php-foundation/ "The New Life of PHP – The PHP Foundation")》。基金会所募集的资金，将会用于资助开发者在 PHP 上工作。

基金会的宣发上，讲的很清楚，为的就是**避免再发生失去 PHP 的主要贡献者**的事情发生，这影响是非常之巨大的。

果然，面包和理想，还是要有权衡，Typora 都开始收费了。

## Rust 社区内讧

在 11 月 22 日，Rust 社区的审核团队集体辞职（Moderation Team Resignation），立即生效。宣布用的 pr 如下图：

![](https://files.mdnice.com/user/3610/a4359504-a128-43e1-9b36-b855307b8825.png)

团队辞职的原因是：**为了抗议 Rust 核心团队将自己置于对任何人都不负责任的地位，只对自己负责**。

一时间，吃瓜众说纷纭。但没有人再给出官方解释了。业内猜测有以下两个观点：
- 在 Rust 运作上：认为与亚马逊正在试图侵蚀 Rust 有关。包括：雇佣了语言团队、编译器团队负责人等。
- 在 Rust 基金会上：亚马逊决定不设立 Rust 基金会 ED，主席将在 Rust 基金会中拥有巨大的权力。

截止 11 月 27 号，这件事感觉已经见不到 “真相” 了。因为 [pr](https://github.com/rust-lang/team/pull/671 "pr") 和 [reddit](https://www.reddit.com/r/rust/comments/qzme1z/moderation_team_resignation/ "reddit") 上的讨论帖均已锁定，没有正式回复。

总感觉有种黑幕的感觉？

## 总结

看了这两起社区重大异常后，再对比看 Go 社区，似乎又比较的温和。毕竟 Go 一开始的诞生，就来源于 Google 大佬们在职期间对既有使用语言的不满。

这么多年了，他们也一直没有离职。Google 也提供了不少的时间和资金给 Go 核心团队做宣传和维护社区，相对安全，还**能定期休假 2 周**（静默期），真是太羡慕了。

但在反面来看，也有很多人嫌弃 Go 背后的靠山是 Google，你怎么看呢？

欢迎在评论区留言和交流：）