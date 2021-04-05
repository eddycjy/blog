---
title: "令人激动！Go 泛型代码合入 master（附尝鲜方法）"
date: 2021-04-05T16:06:50+08:00
images:
tags: 
  - go
---

大家好，我是慢一拍的后方记者煎鱼。

按照先前官方和文章的说法，Go 泛型预计是在 Go1.18 正式释出。

![](https://imgkr2.cn-bj.ufileos.com/8971e01e-75f8-47c2-b9d3-f6b0ebd85d0d.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=8eKquOL0aM7GzJXegZUzGkOmLPg%253D&Expires=1614491040)


在 GopherCon 2020 Go Team AMA 时，要在今年底要能有生产环境的试用版上线，这是 rsc 所提出的一个管理目标。

## 转折点

近期出现了一个新的转折点，能够让大家在主干分支（master）上就能享受到泛型的功能。

![](https://imgkr2.cn-bj.ufileos.com/3acc09a4-d5a2-49e7-b272-b2fa1571f589.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=zXcEIWPbc%252BUFBDmTyYrRVywwKuI%253D&Expires=1614491656)

而 master 分支对应了 Go1.17 的版本。因此未来将可以在 Go1.17 使用到泛型，这是一个比较惊喜的事情。

## 原因

这件事情为什么会突然发生呢？一切都得从背景说起。原本 Go 泛型是一直在 [dev.typeparams](https://github.com/golang/go/tree/dev.typeparams) 分支上进行研讨和开发。

由于泛型不是简单的一两个模块的代码变更，而是涉及大量的代码变更。

因此需要经常保持与 master 分支的代码同步（近两个月共 20+ 次），会涉及代码冲突/合并的处理，且对于一些冲突的模块他们也不熟悉，所以期望迁移到 master 分支上进行开发。

## 如何不影响既有功能

这类提前放入主版本的操作，在 Go 语言中并不少见。像是现在所见的 `GO111MODULE`，早期的 `GO15VENDOREXPERIMENT` 都有些这么个味道。都是逐步入场，分阶段使用，等确定成熟、完善后再渐渐去掉。

因此本次泛型也采取了这种方法，按照提案，目前使用的是 `-G` 标识做为泛型的开关。

计划如下：

- `-G=0`：继续使用传统的类型检查器。
- `-G=1`：使用 type2，但不支持泛型。
- `-G=2`：使用 type2，支持泛型。

在完成 types2 的错误和现有的错误的开发协调后，计划在 Go 1.17 将 `-G=1` 设置为默认值。

未来也许可以在 Go 1.18 中放弃对 `-G=0` 的支持，这样后续在默认启用 `-G=2` 上会变得更容易。

## 在 Go1.17 尝鲜

在 Go1.17 尝鲜，也就意味着需要拉取 Go 语言的 master 分支的代码，Go1.17 现在正处于开发阶段：

![](https://imgkr2.cn-bj.ufileos.com/8690e590-08d0-416b-b7c0-76a3cd9fbd2b.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=YDlj2rmds9pT8iLgaho3cITFCuw%253D&Expires=1614498690)

我们可以通过 `gotip` 来达到下载 master 分支代码的目的：

```
$ go get golang.org/dl/gotip
$ gotip download
From https://go.googlesource.com/go
 * branch            master     -> FETCH_HEAD
   44361140c0..d9fd38e68b master     -> origin/master
Previous HEAD position was 44361140c0 embed: update docs for proposal tweaks
...
```

在拉取完毕后可以执行 `gotip version` 查看所拉取的版本（commit-id）：

```
$ gotip version
go version devel +d9fd38e68b Sat Feb 27 03:03:29 2021 +0000 darwin/amd64
```

在确定 `gotip` 正常后，我们就可以编写泛型的示例代码了，如下：

```
func Print[T any](s []T) {
	for _, v := range s {
		fmt.Print(v)
	}
}

func main() {
	Print([]string{"脑子进, ", "煎鱼了\n"})
}
```

如果执行像往常那样执行，是会直接提示无法识别泛型的一些标识符：

```
$ gotip run main.go 
# command-line-arguments
./main.go:7:6: missing function body
./main.go:7:11: syntax error: unexpected [, expecting (
```

结合上文的解析，我们需要指定 `-G` 标识，就可以运行了。如下：

```
$ gotip run -gcflags=all=-G=3 main.go 
# command-line-arguments
./main.go:7:6: internal compiler error: Cannot export a generic function (yet): Print
```

显然，正确的走进泛型的逻辑里去了，虽然愉快的报错了，但 Matthew Dempsky 表示这很正常，毕竟 Go 泛型还在开发阶段。

可能会有的小伙伴注意到，`-G` 指定的是 3，与前文所述不符。这与早期的编码有关：

![](https://imgkr2.cn-bj.ufileos.com/4599daf1-36ef-4c4a-bdf7-33bcbe7ac1cb.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=S9zQAMxyQpdxryZN%252FrwkYOmqDI4%253D&Expires=1614499432)

已经提了 CL 变更，只是代码冲突了，待解决。

## 总结

Go 语言的泛型开发计划已经比较明确。首先合入 master 分支，再逐步完成开发，逐步开放。

再进行 `-G` 默认值的调整，最后在泛型完善后就默认开启，把 `-G` 标识彻底去掉。

细品，与 Go modules 的方向是不是差不多。一开始 `GO111MODULE` 需要手动开启 `on`（也就是默认 `off`），再到 Go1.16 `GO111MODULE` 默认为 `on`。

以此完成了一个正反馈的循环，逐步开放，接受社区反馈和开发调整。

结论，**Go 泛型指日可待了**。
