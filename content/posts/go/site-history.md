---
title: "分久必合，golang.org 将成为历史！"
date: 2021-12-31T12:55:03+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

这两天看到官方博客的《[Tidying up the Go web experience](https://go.dev/blog/tidy-web "Tidying up the Go web experience")》，已经明确了优化 Go 站点的计划和安排了，为此今天和大家分享这一个好消息。

在之前 Go 官方推出了新的站点 go.dev，一个新的 Go 开发者中心：

![go.dev](https://files.mdnice.com/user/3610/c70f1330-64da-4b65-8988-f9dbe6b1df05.png)

以及提供给开发者查询 Go 包（package）和模块（module）信息的配套网站 pkg.go.dev：

![pkg.go.dev](https://files.mdnice.com/user/3610/41df22df-78e2-4b15-83d7-c8e38fcaba54.png)

看上去似乎美好，但其实原有的 golang.org 在继续提供 Go 发行版下载、文档和标准库的软件包的资料。

而其他 Go 网站，例如：blog.golang.org、play.golang.org、talks.golang.org 以及tour.golang.org，又都拥有额外的 Go 资料。这一切都有些零散和混乱。

在软件发展来讲，这种被称之为 ”绞杀者“ 模式：

![图来自网络](https://files.mdnice.com/user/3610/4645e874-771a-452a-b4ba-e41d8ffe5673.png)

也就是旧系统不断地被新系统取代，在两边都还存在流量时，会保持两边的运行。随着时间的推移，新系统不断地取代老系统，最终完全取代。

Go 官方**计划在接下来的一两个月里，将把 golang.org 网站合并成一个统一的网站**，也就是在 go.dev 上。

所有的现有 URL（例如：Go 标准库、Go 博客等）都会重定向到新的 URL 地址中，不会出现破坏性修改（无法访问），一切按计划都是兼容的。


![](https://files.mdnice.com/user/3610/0708fcf4-90b2-4451-8a3d-5bf5ce2a1dfa.png)


咱们将会有一个更统一的 Go 网站，部分 Go 初学者也不再需要辛辛苦苦再翻山倒海先找个天梯了，因为是**可以直接访问的**，这是一种利好！

不得不感叹语句，Russ Cox 推动事情的能力真是杠杠的，虽然也会引来不少争议和讨论，但我们依然可以从中学习到好的部分！

大家**有没有什么也想让 Go 团队优化的呢，欢迎在评论区交流和讨论**！