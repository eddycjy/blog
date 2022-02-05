---
title: "学会使用 GDB 调试 Go 代码"
date: 2021-12-31T12:54:57+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

上一篇文章《一个 Demo 学会使用 Go Delve 调试》我们详细介绍了 Go 语言如何使用 Delve 进行排查和调试，对于问题的解决非常的有帮助。

但调试工具肯定不止只有 Delve，今天我们来介绍第二个神器，那就是：GDB。

## GDB 是什么

GDB 是一个类 UNIX 系统下的程序调试工具，允许你看到另一个程序在执行时 "内部 "发生了什么，或者程序在崩溃时正在做什么。

![GDB Logo](https://files.mdnice.com/user/3610/35fc6475-ec0d-44a0-ae8c-920121130edb.png)

主要可以做四类事情：

1. 启动你的程序，指定任何可能影响其行为的东西。
2. 使你的程序在指定的条件下停止。
3. 检查当你的程序停止时发生了什么。
4. 改变你程序中的东西，这样你就可以试验纠正一个错误的影响，并继续了解另一个错误。

## 安装

如果是在 MacOS 上的话，可以直接使用 brew 安装：

```
brew install gdb
```

如果是在 Linux ，则使用自带的包管理工具进行安装即可，但需要注意安装完毕后需要在 HOME 目录进行相关配置。

安装完毕后，执行 `gdb` 就可以看到：

```
$ gdb
GNU gdb (GDB) 10.2
...
(gdb) 
```

写此文时最新的 gdb 版本已经是 10.2 了，我也升级了上去。问题不大，还多了不少功能。

## 编译

我们还是使用先前的演示程序来进行调试。但由于 Go 语言的不少编译优化，因此在编译运行程序时，有以下几点需要注意：

- go build 编译时需要增加 `-gcflags=all="-N -l"` 指令来关闭内联优化，方便接下来的调试。

- 若是 MacOS，在 go build 编译时需要增加 `-ldflags='-compressdwarf=false'` 指令。
  - 若不禁止，则会出现 `No symbol table is loaded. Use the "file" command.` 的错误。
  - Go 编译默认为了减少二进制大小会默认压缩 DWARF 调试信息，但这会影响 gdb 的调试，因此需要将其关闭。
  
编译的命令是：

```
$ go build -gcflags=all="-N -l" -ldflags='-compressdwarf=false' .
```

输出结果：

```
！了鱼煎进子脑
```
  
## 尝试 gdb

GDB 有两种调试模式，分别是文本用户界面（Text User Interface，简称 tui）和默认的命令行模式：

```
// 调试界面
$ gdb -tui ./awesome-project

// 命令行模式
$ gdb ./awesome-project
```

接下来我们使用 gdb tui 的调试模式来给大家演示功能。

我们在执行命令 `gdb -tui ./awesome-project` 后，窗口会切换为如下：


![gdb tui 初始样子](https://files.mdnice.com/user/3610/2a7096f4-addd-404f-9a47-4a97cbe9d595.png)

你会发现中间提示 “No Source Available”，此时你需要继续回车两次，他就会自动加载插件支持，提示：“Loading Go Runtime support.”。

我们就可以看到具体的代码块内容，如下：

![](https://files.mdnice.com/user/3610/351571a3-b89e-42f2-941e-24e734bfae26.png)

用 MacOS 的同学需要注意，如果你在断点时发现发现了如下错误：

```
(gdb) b main.main
Breakpoint 1 at 0x10a2ea0: file /Users/eddycjy/go-application/awesomeProject/main.go, line 15.
(gdb) r
Starting program: /Users/eddycjy/go-application/awesomeProject/hello
Unable to find Mach task port for process-id 64212: (os/kern) failure (0x5).
 (please check gdb is codesigned - see taskgated(8))
```

也就是 “please check gdb is codesigned - see taskgated(8)”，则需要重新处理证书认证和授权，是 MacOS 使用上的一个问题，具体可参考：《[Codesign gdb on OSX](https://gist.github.com/hlissner/898b7dfc0a3b63824a70e15cd0180154)》。

解决后，咱们的 gdb 就算是能够正确的运行起来了！

## 常用 gdb 命令

在 gdb 中，和 dlv 一样有常用的关键字命令。当然了，gdb 的 help all 输出非常多：

```
(gdb) help all

Command class: aliases
Command class: breakpoints

awatch -- Set a watchpoint for an expression.
break, brea, bre, br, b -- Set breakpoint at specified location.
break-range -- Set a breakpoint for an address range.
catch -- Set catchpoints to catch events.
...
```

常用的关键字如下：
- b：break 的缩写，作用是打断点，例如：main.main，可带代码行数。
- r：run 的缩写，作用是运行程序到下一个断点处。
- c：continue 的缩写，作用是继续执行到下一个断点。
- s：step 的缩写，作用是单步执行，如果有所调用的方法，将会进入该方法。
- l：list 的缩写，作用是查看对应的源码。
- n：next 的缩写，作用是单步执行，不会进入所调用的方法，。
- q：quit 的缩写，作用是退出。
- info breakpoints：作用是查看所有设置的断点信息。
- info locals：作用是查看变量信息。
- info args：作用是查看函数的入参和出参的具体值。
- info goroutines：作用是查看 goroutines 的信息。
- goroutine 1 bt：作用是查看指定序号的 goroutine 调用堆栈。

## 进行调试

在调试上与 dlv 差不多，也是先执行关键字 `b` 打断点：

```
(gdb) b main.main
Breakpoint 1 at 0x10cbaa0: file /Users/eddycjy/go-application/awesomeProject/main.go, line 9.
```

也可以先执行关键字 `l` 查看对应的代码情况再进行做决定：

```
(gdb) l main.main
4		"fmt"
5	
6		"github.com/eddycjy/awesome-project/stringer"
7	)
8	
9	func main() {
10		fmt.Println(stringer.Reverse("脑子进煎鱼了！"))
11	}
```

查看对应 goroutines 正在运行的函数情况：

```
(gdb) info goroutines
  1  waiting runtime.gosched
* 13  running runtime.goexit
```

根据 pprof 等所得到的 goroutine 序号进行进一步的分析：

```
(gdb) goroutine 1 bt
#0  0x000000000040facb in runtime.gosched () at /home/user/go/src/runtime/proc.c:873
#1  0x00000000004031c9 in runtime.chanrecv (c=void, ep=void, selected=void, received=void)
 at  /home/user/go/src/runtime/chan.c:342
#2  0x0000000000403299 in runtime.chanrecv1 (t=void, c=void) at/home/user/go/src/runtime/chan.c:423
#3  0x000000000043075b in testing.RunTests (matchString...
```

注意一个细节，gdb 调试是可以看到并对 runtime 包内容的代码进行断点和分析的。

也可以和 dlv 一样执行 p 关键字输出相应的值的类型、值内容：

```
(gdb) p re
(gdb) p t
$1 = (struct testing.T *) 0xf840688b60
(gdb) p t
$1 = (struct testing.T *) 0xf840688b60
(gdb) p *t
$2 = {errors = "", failed = false, ch = 0xf8406f5690}
(gdb) p *t->ch
$3 = struct hchan<*testing.T>
```

与 dlv 大同小异。

## 总结

总体上来讲，MacOS 上使用 gdb 还是挺麻烦的，在 Linux 环境下使用 gdb 还是更方便些。

由于 **dlv 和 gdb 在大致的调试上不会差距的太远**，因此本文就没有过于展开。

若是对业务代码进行分析，更建议使用 dlv，也就是我们上一篇文章所讲的内容。**若有 runtime 库的调试需求的话，推荐使用 gdb 来作为首要调试工具**，若无这方面诉求，建议使用 dlv。