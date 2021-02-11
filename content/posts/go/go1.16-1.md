---
title: "Go1.16 即将正式发布，以下变更你需要知道"
date: 2021-02-11T16:13:15+08:00
images:
tags: 
  - go
---

大家好，我是正在努力学习的煎鱼。

在前几天，Go1.16rc1 抢先发布了。结合常规的 28 发布规律，其将会在 2021.02 月份左右发布正式版本。

![](https://imgkr2.cn-bj.ufileos.com/417219ea-578e-48f0-8a83-84544000a698.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=mbLHd49lXRyOoY%252FlyamD5YnGagw%253D&Expires=1612064951)

这次 Go1.16 也带来了一些新特性或变更。那么作为一个 Gopher，想必不能错过这次的更新。

![](https://imgkr2.cn-bj.ufileos.com/4161e3cd-9f66-4f3f-8f68-91f3ba496c18.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=CwpOhSJnk70gEkTF8ITIxC58PoM%253D&Expires=1612074386)

今天这篇文章将会带大家了解一下 Go1.16 的几个需要关注的特性。

### 废弃 io/ioutil

Go 官方认为 io/ioutil 这个包的定义不明确且难以理解。所以 Russ Cox 在 2020.10.17 提出了废弃 io/ioutil 的提案。

大致变更如下：
- Discard => io.Discard
- NopCloser => io.NopCloser
- ReadAll => io.ReadAll
- ReadDir => os.ReadDir
- ReadFile => os.ReadFile
- TempDir => os.MkdirTemp
- TempFile => os.CreateTemp
- WriteFile => os.WriteFile

与此同时大家也不需要担心存在破坏性变更，因为有 Go1 兼容性的保证，在 Go1 中 io/ioutil 还会存在，只变更内部的方法调用：

```
func ReadAll(r io.Reader) ([]byte, error) {
    return io.ReadAll(r)
}

func ReadFile(filename string) ([]byte, error) {
    return os.ReadFile(filename)
}
```

大家在后续也可以改改调用习惯。

### 支持静态资源嵌入

如果我们希望把静态文件编译进 Go 的二进制文件的话，在以往需要借助 go-bindata/go-bindata 这类第三方开源库来实现。

而从 Go1.16 起，通过 `go:embed` 就可以快速实现这个功能：

```
import _ "embed"

//go:embed hello.txt
var s string
print(s)
```

通过对变量 `s` 声明 `go:embed` 指令，使其在编译时读取当前目录下的 `hello.txt` 文件。

最终变量 `s` 就会输出 `hello.txt` 文件中的字符串内容。 

### 新增 io/fs 的支持 

新增了标准库 io/fs，正式将文件系统相关的基础接口抽象到了该标准库中。

以前的话大多是在 `os` 标准库中，这一步抽离更进一步的抽象了文件树的接口。在后续的版本中，大家可以优先考虑使用 `io/fs` 标准库。

### 调整切片扩容策略

Go1.16 以前的 slice 的扩容条件是 `len`，在最新的代码中，已经改为了以 `cap` 属性作为基准：

```
  // src/runtime/slice.go
	if cap > doublecap {
		newcap = cap
	} else {
		// 这是以前的代码：if old.len < 1024 {
		// 下面是 Go1.16rc1 的代码
		if old.cap < 1024 {
			newcap = doublecap
		}
```

以官方的 test case 为例：

```
func main() {
	const N = 1024
	var a [N]int
	x := cap(append(a[:N-1:N], 9, 9))
	y := cap(append(a[:N:N], 9))
	println(cap(x), cap(y))
}
```

在 Go1.16 以前输出 2048, 1280。在 Go1.16 及以后输出 1280, 1280，保证了两种的一致。

### 支持 Apple Silicon M1

众所周知，最新版本的 Mac 采用了新的 64 位 ARM 架构，因此在 Go1.16 后正式支持了 `GOOS=darwin` 和 `GOARCH=arm64`。

而相应的先前用于 iOS 端口的，将改为 `GOOS=ios` 和 `GOARCH=arm64`。

同时 Apple M1 能不能很好的跑好 Go 语言程序也是各大微信群爱讨论的问题，在 GoLand 上：

![图来自网络，路过微信群看见](https://static01.imgkr.com/temp/960978d546694a5c94ca2af70e44c83c.png)

需要注意，GoLand 的一些给你要到后续的新版本才可以使用。

### 调整 Go modules 策略

从 Go1.16 起，Go modules 的环境变量 `GO111MODULE `默认开关将为 `on`，不再是之前是 `auto` 了。

还在使用 GOPATH，或 Go modules 没切全的同学这一块需要特别注意。

### 新增 GODEBUG inittrace

GODEBUG 新增 inittrace 指令，可以用于 `init` 方法的排查：

```
$ GODEBUG=inittrace=1 go run main.go 
```

输出结果：

```
init internal/bytealg @0.008 ms, 0 ms clock, 0 bytes, 0 allocs
init runtime @0.059 ms, 0.026 ms clock, 0 bytes, 0 allocs
init math @0.19 ms, 0.001 ms clock, 0 bytes, 0 allocs
init errors @0.22 ms, 0.004 ms clock, 0 bytes, 0 allocs
init strconv @0.24 ms, 0.002 ms clock, 32 bytes, 2 allocs
init sync @0.28 ms, 0.003 ms clock, 16 bytes, 1 allocs
init unicode @0.44 ms, 0.11 ms clock, 23328 bytes, 24 allocs
...
```

主要作用是 init 函数跟踪的支持，以用于 init 调试和启动时间的概要分析，算是一个 GODEBUG 的补充功能点。

## 简化结构体标签

在 Go 语言的结构体中，我们常常会因为各种库的诉求，需要对结构体的 `tag` 设置标识。

如果像是以前，量比较多就会变成：

```
type MyStruct struct {
  Field1 string `json:"field_1,omitempty" bson:"field_1,omitempty" xml:"field_1,omitempty" form:"field_1,omitempty" other:"value"`
}
```

但在 Go1.16 及以后，就可以通过合并的方式：

```
type MyStruct struct {
  Field1 string `json,bson,xml,form:"field_1,omitempty" other:"value"`
}
```

方便和简洁了不少。

## 总结

在本次 Go1.16 中带来了不少小优化和新的特性支持。离 Go1.18 的泛型又近了一步。

另外在本次新版本中，像是 `template` 支持跨行：

```
{{"hello" |
   printf}}
```

又或是 Linux 的默认内存管理策略下又从 MADV_FREE 改回了 MADV_DONTNEED 策略，大家在新版本中不再需要设置：

```
GODEBUG=madvdontneed=1
```

大家若有需求都可以进一步去了解，现在新版本的功能特性已经锁定，基本尘埃落定。

传送门：https://tip.golang.org/doc/go1.16。



