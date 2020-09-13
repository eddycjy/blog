---
title: "Go: A Documentary 发布！"
date: 2020-09-11T20:46:38+08:00
toc: false
images:
tags: 
  - untagged
---

以前经常有读者问我，哪儿可以找到 Go 语言的前世今生，这种时候我们往往会告诉他去看 issues 和 proposals。但资料有点分散，且没有索引体系。因此不少人新入门的读者读着读着就跑偏了，又或是在第一步找资料上就被拦住了。

最近欧神（@changkun）低调的发布了 《Go: A Documentary》，这个文档收集了 Go 开发过程中许多有趣（公开可见的）的问题，讨论，提案，CL 和演讲，其目的是为 Go 历史提供全面的参考。

个人认为这份资料非常的有价值，相当于欧神把资料索引整理好了，强烈推荐对 Go 语言感兴趣的读者进行阅读：

![image](https://image.eddycjy.com/0643ae29cdb8fb99d5a83bf67b443a9a.jpg)

内容索引主要分为：

- Sources
- Committers
    - Core Authors
    - Compiler/Runtime Team
    - Library/Tools/Security/Community
    - Group Interviews
- Timeline
- Language Design
    - Misc
    - Slice
    - Package Management (1.4, 1.5, 1.7)
    - Type alias (1.9)
    - Defer (1.13)
    - Error values (1.13)
    - Channel/Select
    - Generics
- Compiler Toolchain
    - Compiler
    - Linker
    - Debugger
    - Tracer
    - Builder
    - Modules
    - gopls
    - Testing
- Runtime Core
    - Statistics
    - Scheduler
    - Execution Stack
    - Memory Allocator
    - Garbage Collector
    - Memory model
    - ABI
- Standard Library
    - syscall
    - io
    - go/*
    - sync
    - Pool
    - Mutex
    - atomic
    - time
    - context
    - encoding
    - image, x/image
    - misc
- Unclassified But Relevant Links
- Fun Facts
- Acknowledgements

《Go: A Documentary》 的访问地址是 `https://golang.design/history/`，GitHub 仓库地址：`https://github.com/golang-design/history`，大家也可以通过 “阅读原文” 进入。
