---
title: "Go1.17 新特性，优化抛出的错误堆栈"
date: 2021-12-31T12:55:05+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

平时在日常工程中，我们常常会用到异常恐慌（panic）的记录和追踪。最常见的就是，线上 panic 了之后，我们总想从中找到一些蛛丝马迹。

我们很多人是看 panic 是看他的调用堆栈。然后就开始猜，看代码。猜测是不是哪里写的有问题，就想知道 panic 是由什么参数引起的？

因为知道了诱发的参数，排查问题就非常方便了。为此**在 Go1.17，官方对这块的调用堆栈信息展示进行了优化**，使其可读性更友好。

## 案例

结合我们平时所使用的 panic 案例。如下：

```golang
func main() {
	example(make([]string, 1, 2), "煎鱼", 3)
}

//go:noinline
func example(slice []string, str string, i int) error {
	panic("脑子进煎鱼了")
}
```

运行结果：

```
$ go run main.go
panic: 脑子进煎鱼了

goroutine 1 [running]:
main.example(0xc000032758, 0x1, 0x2, 0x1073d11, 0x6, 0x3, 0xc000102058, 0x1013201)
	/Users/eddycjy/go-application/awesomeProject/main.go:9 +0x39
main.main()
	/Users/eddycjy/go-application/awesomeProject/main.go:4 +0x68
exit status 2
```

我们函数的入参是：`[]string、string、int`，核心关注到 `main.example` 方法的调用堆栈信息：

```golang
main.example(
    0xc000032758, 
    0x1, 
    0x2, 
    0x1073d11, 
    0x6, 
    0x3, 
    0xc000102058, 
    0x1013201
)
```
明明只是函数三个参数，却输出了一堆，对应起来非常的不清晰。

其实际对应是：

- slice：0xc000032758、0x1、0x2。
- string：0x1073d11、0x6。
- int：0x3。

这里存在的问题是，看调用堆栈的人，还得必须了解基本数据结构（例如：slice、string、int 等），他才知道每个函数入参他对应拥有几个字段，才能知道其内存布局的结构，有一点麻烦。

并且从程序运行的角度来讲，这么水平平铺的方式，并不直观和准确。因为不同类型他是多个字段组合成结构才能代表一个类型。这不得还要人为估测？

## 优化

终于，这一块的调用堆栈查看在 Go1.17 做了正式的改善。如下：

```golang
$ go1.17 run main.go 
panic: 脑子进煎鱼了

goroutine 1 [running]:
main.example({0x0, 0xc0000001a0, 0xc000034770}, {0x1004319, 0x60}, 0x0)
	/Users/eddycjy/go-application/awesomeProject/main.go:9 +0x27
main.main()
	/Users/eddycjy/go-application/awesomeProject/main.go:4 +0x47
exit status 2
```

新版本的调用堆栈的信息改变：

```golang
main.example(
    {0x0, 0xc0000001a0, 0xc000034770}, 
    {0x1004319, 0x60}, 
    0x0
)
```

在 Go 语言以前的版本中，调用堆栈中的函数参数被打印成基于内存布局的十六进制值的形式，比较难以读取。
 
在 **Go1.17 后，每个函数的参数都会被单独打印，并且以 “，” 隔开**，复合数据类型（例如：结构体、数组、切片等）的参数会用大括号包裹起来，整体更易读。

其实际对应如下：

- slice：0x0, 0xc0000001a0, 0xc000034770。
- string：0x1004319, 0x60。
- int：0x0。

这里也有一块细节要注意，你会发现 Go1.17 的函数参数的数量和以往的版本相比，少了。是因为函数的返回值存在于寄存器中，而不会存储到内存中。

因此函数返回值可能会是不准确的，所以也在新版本中也就不再打印了。

## 总结

在 Go1.17 的新版本中，调用堆栈的函数参数的可读性得到了进一步的优化和调整，在后续的使用上可能能够带来一定的排错效率的提高。

你平时在借助调用堆栈排查问题呢，希望还获得什么辅助呢？

## 参考
- [GoTip: New Stack Trace Output Wrong](https://github.com/golang/go/issues/46708)
- [cmd/compile: bad traceback arguments](https://github.com/golang/go/issues/45728)
- [Go 1.17新特性详解：使用基于寄存器的调用惯例](https://mp.weixin.qq.com/s/AkJoXLlpSmw5vMZDpXoq5w)
- [doc/go1.17: reword "results" in stack trace printing](https://groups.google.com/g/golang-codereviews/c/JkhaLqHFReM?pli=1)