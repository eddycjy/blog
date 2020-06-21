---
title: "为什么容器内存占用居高不下，频频 OOM（续）"
date: 2020-06-19T21:29:08+08:00
draft: true
toc: true
tags: 
  - go
---

在上周的文章[《为什么容器内存占用居高不下，频频 OOM》](/posts/why-container-memory-exceed/) 中，我根据现状进行了分析和说明，收到了很多读者的建议和疑惑，因此有了这一篇文章，包含更进一步的说明和排查。

## 疑问

一般我们可以通过 `free -m` 查看当前系统的内存使用情况：

![image](https://image.eddycjy.com/daf2a1d53f4bf0f21e315d2333e08159.png)

在发现是系统内存占用高后，就会有读者会提到，为什么不 “手动清理 Cache”，因为 Cache 高的话，可以通过下述命令来清理：

1. 清理 page cache：

```
$ echo 1 > /proc/sys/vm/drop_caches
```

2. 清理 dentries 和 inodes：

```
$ echo 2 > /proc/sys/vm/drop_caches
```

3. 清理 page cache、dentries 和 inodes：

```shell
$ echo 3 > /proc/sys/vm/drop_caches
```

但新问题又出现了，因为我们的命题是在容器中，在 Kubernetes 中，若执行 drop_caches 相关命令，将会对 Node 节点上的所有其他应用程序产生负面影响，尤其是那些占用大量 IO 并由于缓冲区高速缓存而获得更好性能的应用程序。

我想这并不是一个好办法。

## 表象

为什么要排查这个问题，原因就是容器设置了 Memory Limits，而达到了 Limits 上限，被 OOM 掉了。所以我们想知道为什么会出现这个情况，而在前文中我们针对了五大类情况进行了猜想：

- 频繁申请重复对象
- 不知名内存泄露
- madvise 策略变更
- 监控/判别条件有问题
- 容器环境的机制

最终判定其容器 Memory OOM 判定标准是 container_memory_working_set_bytes 指标，其实际组成为 RSS + Cache（最近访问的内存、脏内存和内核内存）。

此时我们可以通过 Linux pmap 来查看该容器进程的内存映射情况：

![image](https://image.eddycjy.com/0bb82eabe1fcc1a5a65f4382932a6d2c.jpg)

我们发现了大量的 mapping 为 anon 的内存映射，最终 totals 确实达到了容器 Memory 相当的量，那么 anon 又是什么呢，anon 行表示在磁盘上没有对应的文件，也就是没有实际存在的载体，是 anonymous。

## 思考

既然存在如此多的 anon，结合先前的考虑，我们知道出现这种情况的服务都是文件处理型服务，包含大量的批量生成图片、生成 PDF 等资源消耗型的任务，也就是会瞬间申请大量的内存，使得系统的空闲内存触及全局最低水位线（global wmark_min），而在触及全局最低水位线后，会尝试进行回收，实在不行才会触发 Cgroup OOM 的行为。

那么更进一步思考的是两个问题，一个是 Cgroup 达到 Limits 前的尝试释放仍然不足以支撑所需申请的连续内存段，而另外一个问题就是为什么 Cache 并没有释放：

![image](https://image.eddycjy.com/2e6c8c153836b29175dff7623ec67a0a.png)

通过上图，可以肯定的该服务在凌晨 00：00-06：00 是没有什么流量的，但是 container_memory_working_set_bytes 指标依旧稳定不变，排除 RSS 的原因，那配合指标的查看基本确定是 Cache 没有释放。

而 Cache 的出现，在该服务中大多数应该是由于其频繁操作文件导致，因为在 Linux 中，在第一次读取文件时会将一份放到系统 Cache，另外一份则放入进程内存中使用。关键点在于当进程运行完毕关闭后，系统 Cache 是不会马上回收的，需要经过系统的内存管理后再适时被释放。

又关注到表象，现在我们发现 Cache 的持续不释放，进程也没外部流量，RSS 又低的可怜，Cache 不像 hold 住了的样子，毕竟系统 Cache 多用于在下次访问时提供的高速访问。

最终就考虑到是否 Linux 内核在这块内存管理上存在 BUG 呢？

## 根因

### 问题版本

该服务所使用的 Kubernetes 是 1.11.5 版本，Linux 内核版本为 3.10.x，时间为 2017 年 9 月：

```
$ uname -a
Linux xxxxx-xxx-99bd5776f-k9t8z 3.10.0-693.2.2.el7.x86_64 #1 SMP Tue Sep 12 22:26:13 UTC 2017 x86_64 Linux
```

都算是有一定年代的老版本了。

### 原因分析

memcg 是 Linux 内核中管理 cgroup 内存的模块，但实际上在 Linux 3.10.x 的低内核版本中存在不少实现上的 BUG，其中最具代表性的是 memory cgroup 中 kmem accounting 相关的问题（在低版本中属于 alpha 特性）：

- slab 泄露：具体可详见该文章 [SLUB: Unable to allocate memory on node -1](https://pingcap.com/blog/try-to-fix-two-linux-kernel-bugs-while-testing-tidb-operator-in-k8s/#bug-1-unstable-kmem-accounting) 中的介绍和说明。

- memory cgroup 泄露：在删除容器后没有回收完全，而 Linux 内核对 memory cgroup 的总数限制是 65535 个，若频繁创建删除开启了 kmem 的 cgroup，就会导致无法再创建新的 memory cgroup。

当然，为什么出现问题后绝大多数是由 Kubernetes、Docker 的相关使用者发现的呢，这与云原生的兴起，这类问题与内部容器化的机制相互影响，最终开发者 “发现” 了这类应用频繁出现 OOM。

## 解决方案

### 调整内核参数

关闭 kmem accounting：

```
cgroup.memory=nokmem
```

也可以通过 kubelet 的 nokmem Build Tags 来编译解决：

```
$ kubelet GOFLAGS="-tags=nokmem"
```

但需要注意，kubelet 版本需要在 v1.14 及以上。

### 升级内核版本

升级 Linux 内核至 kernel-3.10.0-1075.el7 及以上就可以修复这个问题，详细可见 [slab leak causing a crash when using kmem control group](https://bugzilla.redhat.com/show_bug.cgi?id=1507149#c101)，其在发行版中 CentOS 7.8 已经发布。

## 总结

在写下这篇文章时，我们可以看到 kmem accounting 的不少问题都已经被修复或提上日程，这对本次排查提供了相当大的便利，在确定问题的所在后根据 cgroup leak 沿着排查下去，基本都能看到大量的前人所经历过的 “挣扎”，大家若有兴趣，也可以跟着参考所提供的的链接做更一进步的深入了解。

但事实上，不管哪个 Linux 内核版本，都存在着或多或少的问题，若赶新，需要做好适当的心理准备，否则就会遇到 “没上容器时好好的” 的窘境，查起问题更麻烦，因为不少人是没有产线权限的。