---
title: "Go 1.17 支持泛型了？具体怎么用"
date: 2021-12-31T12:55:02+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

千呼万唤的，Go1.17 前几天终于发布了：

![](https://files.mdnice.com/user/3610/45fe4f8a-ef8e-41a0-abba-ecdc05fe61c5.png)

先前我写了几篇 Go1.17 新特性的文章，有兴趣的小伙伴可以看看：

- [一个新细节，Go 1.17 将允许切片转换为数组指针！](https://mp.weixin.qq.com/s/v1czjzlUsaSQTpAOG9Ub3w)
- [我要提高 Go 程序健壮性，Fuzzing 来了！](https://mp.weixin.qq.com/s/zdsrmlwVR0bP1Q_Xg_VlpQ)
- [提了 3 年，Go1.17 终于增强构建约束！](https://mp.weixin.qq.com/s/5kLFIuI0UJl_o8vMmZNfoA)

今天的主题是泛型，众所皆知 Go1.18 泛型就会正式释出，都很期待，毕竟大更新，所有配套都会陆续有来！
其实，**在 Go1.17 的此刻其实可以使用泛型了**，泛型代码已合入 master 分支。

咱们只需要一点点操作，就能提前过上 Go 泛型的实验生活了。

## 升级 Go1.17 

你需要先升级 Go1.17，如下图：

![](https://files.mdnice.com/user/3610/a3260cf0-f9bc-4c86-aafd-4d08480143b9.png)

安装后查看版本信息是否正常输出：

```
go1.17 version
go version go1.17 darwin/amd64
```

## 使用泛型

接着写入一个基本的泛型 Demo：

```golang
import (
	"fmt"
)

func Print[T any](s []T) {
	for _, v := range s {
		fmt.Print(v)
	}
}

func main() {
	Print([]string{"你好, ", "脑子进了煎鱼\n"})
	Print([]int64{1, 2, 3})
}
```

只需要在 run 和 build 的命令执行时指定 `-G` 标识就好了。不过有的小伙伴可能会疑惑，为什么要这么干？

其实这类提前放入主版本的操作，在 Go 语言中并不少见。像是现在所见的 `GO111MODULE`，早期的 `GO15VENDOREXPERIMENT` 都有些这么个味道。都是逐步入场，分阶段使用，等确定成熟、完善后再渐渐去掉。

本次泛型也采取了这种方法，按照提案，目前使用的是 `-G` 标识做为泛型的开关。

运行的命令如下：

```
go1.17 run -gcflags=-G=3 xxx
``` 

就可以运行带有泛型的代码。

查看输出结果：

```
$ go1.17 run -gcflags=-G=3 generics.go
# command-line-arguments
./generics.go:7:6: internal compiler error: Cannot export a generic function (yet): Print

Please file a bug report including a short program that triggers the error.
https://golang.org/issue/new
```

竟然报错了，煎鱼你翻车了是吧...

根据错误提示可得知，是还没实现导出一个通用函数的功能。那样我们只需要把 `Print` 方法改为 `print`，再执行就可以了。

再次执行后的输出结果：

```
你好, 脑子进了煎鱼
123
```
成功输出了不同类型的值。

## 更多的案例

在 GitHub 有个小伙伴 mattn 整理了完整的泛型使用案例后开源了，可以实际下载使用看看：

![github.com/mattn/go-generics-example](https://files.mdnice.com/user/3610/a3e768b4-ca68-448d-b4b8-98a3fce447da.png)

大家根据上面的介绍来实际使用就可以达到运行泛型的效果了，GitHub 地址是：github.com/mattn/go-generics-example。

## 总结

经过多年的折腾，Go 语言在发布的 1.17 版本中已经包含了泛型的功能。将会在 Go1.18 正式宣发泛型，我们将会是Go 历史新阶段的见证者。

为什么？因为随着 Go1.18 的逼近，我们将会将会见到越来越多的新工具支持和变更，甚至会改变不少 Go 工程的写法。

欢迎大家在评论区分享你的看法！