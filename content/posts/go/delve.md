---
title: "一个 Demo 学会使用 Go Delve 调试"
date: 2021-12-31T12:54:57+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

在 Go 语言中，除了 go tool 工具链中的 pprof、trace 等剖析工具的大利器外。常常还会有小伙伴问，有没有更好用，更精细的，

大家总嫌弃 pprof、trace 等工具，不够细，没法一口气看到根因，或者具体变量...希望能够最好能追到代码级别调试的，看到具体变量的值是怎么样的，随意想怎么看怎么看的那种。

为此今天给大家介绍 Go 语言强大的 Delve （dlv）调试工具，来更深入问题剖析。

## 安装

我们需要先安装 Go delve，若是 Go1.16 及以后的版本，可以直接执行下述命令安装：

```
$ go install github.com/go-delve/delve/cmd/dlv@latest
```

也可以通过 git clone 的方式安装：

```
$ git clone https://github.com/go-delve/delve
$ cd delve
$ go install github.com/go-delve/delve/cmd/dlv
```

在安装完毕后，我们执行 `dlv version` 命令，查看安装情况：

```
$ dlv version
Delve Debugger
Version: 1.7.0
Build: $Id: e353a65161e6ed74952b96bbb62ebfc56090832b $
```

可以明确看到我们所安装的版本是 v1.7.0。

## 演示程序

我们计划用一个反转字符串的演示程序来进行 Go 程序的调试。第一部分先是完成 `stringer` 包的 `Reverse` 方法。

代码如下：

```golang
package stringer

func Reverse(s string) string {
	r := []rune(s)
	for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r)
}
```

再在具体的 `main` 启动函数中进行调用。代码如下：

```golang
import (
	"fmt"

	"github.com/eddycjy/awesome-project/stringer"
)

func main() {
	fmt.Println(stringer.Reverse("脑子进煎鱼了！"))
}
```

输出结果：

```
！了鱼煎进子脑
```

## 进行调试

Delve 是 Go 程序的源代码级调试器。Delve 使您能够通过控制流程的执行与您的程序进行交互，查看变量，提供线程、goroutine、CPU 状态等信息。

其一共支持如下 11 个子命令：

```
Available Commands:
  attach      Attach to running process and begin debugging.
  connect     Connect to a headless debug server.
  core        Examine a core dump.
  dap         [EXPERIMENTAL] Starts a TCP server communicating via Debug Adaptor Protocol (DAP).
  debug       Compile and begin debugging main package in current directory, or the package specified.
  exec        Execute a precompiled binary, and begin a debug session.
  help        Help about any command
  run         Deprecated command. Use 'debug' instead.
  test        Compile test binary and begin debugging program.
  trace       Compile and begin tracing program.
  version     Prints version.
```

我们今天主要用到的是 debug 命令，他能够编译并开始调试当前目录下的主包，或指定的包，是最常用的功能之一。

接下来我们利用这个演示程序来进行 dlv 的深入调试和应用。

执行如下命令：

```
➜  awesomeProject dlv debug .
Type 'help' for list of commands.
(dlv) 
```

我们先在演示程序根目录下执行了 debug，进入了 dlv 的交互模式。

再使用关键字 `b`（break 的缩写）对 `main.main` 方法设置断点：

```
(dlv) b main.main
Breakpoint 1 (enabled) set at 0x10cbab3 for main.main() ./main.go:9
(dlv) 
```

设置完毕后，我们可以看到方法对应的文件名、行数。接着我们可以执行关键字 `c`（continue 的缩写）跳转到下一个断点处：

```
(dlv) c
> main.main() ./main.go:9 (hits goroutine(1):1 total:1) (PC: 0x10cbab3)
     4:		"fmt"
     5:	
     6:		"github.com/eddycjy/awesome-project/stringer"
     7:	)
     8:	
=>   9:	func main() {
    10:		fmt.Println(stringer.Reverse("脑子进煎鱼了！"))
    11:	}
(dlv) 
```

在断点处，我看可以看到具体的代码块、goroutine、CPU 寄存器地址等运行时信息。

紧接着执行关键字 `n`（next 的缩写）单步执行程序的下一步：

```
(dlv) n
> main.main() ./main.go:10 (PC: 0x10cbac1)
     5:	
     6:		"github.com/eddycjy/awesome-project/stringer"
     7:	)
     8:	
     9:	func main() {
=>  10:		fmt.Println(stringer.Reverse("脑子进煎鱼了！"))
    11:	}
```

我们可以看到程序走到了 main.go 文件中的第 10 行中，并且调用了 `stringer.Reverse` 方法去处理。

此时我们可以执行关键字 `s`（step 的关键字）进入到这个函数中去继续调试：

```
(dlv) s
> github.com/eddycjy/awesome-project/stringer.Reverse() ./stringer/string.go:3 (PC: 0x10cb87b)
     1:	package stringer
     2:	
=>   3:	func Reverse(s string) string {
     4:		r := []rune(s)
     5:		for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
     6:			r[i], r[j] = r[j], r[i]
     7:		}
     8:		return string(r)
```

输入后，调试的光标会到 `Reverse` 方法上，此时我们可以调用关键字 `p`（print 的缩写）传出所传入的变量的值：

```
(dlv) p s
"脑子进煎鱼了！"
```

此处函数的形参变量是 s，输出了 “脑子进煎鱼了！”，与我们所传入的是一致的。

但故事一般没有这么的简单，会用到 Delve 来调试，说明是比较细致、隐患的 BUG。为此我们大多需要更进一步的深入。

我们继续围观 `Reverse` 方法：

```
     5:		for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
     6:			r[i], r[j] = r[j], r[i]
     7:		}
```

从表现来看，我们常常会怀疑是第 6 行可能是问题的所在。这时可以针对性的对第 6 行进行断点查看：

```
(dlv) b 6
Breakpoint 2 (enabled) set at 0x10cb92c for github.com/eddycjy/awesome-project/stringer.Reverse() ./stringer/string.go:6
```

设置完断点后，我们只需要执行关键字 `c`，继续下一步：

```
(dlv) c
> github.com/eddycjy/awesome-project/stringer.Reverse() ./stringer/string.go:6 (hits goroutine(1):1 total:1) (PC: 0x10cb92c)
     1:	package stringer
     2:	
     3:	func Reverse(s string) string {
     4:		r := []rune(s)
     5:		for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
=>   6:			r[i], r[j] = r[j], r[i]
     7:		}
     8:		return string(r)
     9:	}
```

走到对应的代码片段后，执行关键字 `locals`：

```
(dlv) locals
r = []int32 len: 7, cap: 32, [...]
j = 6
i = 0
```

我们就可以看到对应的变量 r, i, j 的值是多少，可以根据此来分析程序流转是否与我们预想的一致。

另外也可以调用关键字 `set` 去针对特定变量设置期望的值：

```
(dlv) set i = 1
(dlv) locals
r = []int32 len: 7, cap: 32, [...]
j = 6
i = 1
```

设置后，若还需要继续排查，可以继续调用关键字 `c` 去定位，这种常用于特定变量的特定值的异常，这样一设置一调试基本就能排查出来了。

在排查完毕后，我们可以执行关键字 `r`（reset 的缩写）：

```
(dlv)  r
Process restarted with PID 56614
```

执行完毕后，整个调试就会重置，像是前面在打断点时所设置的变量值就会恢复。

若要查看设置的断点情况，也可以执行关键字 `bp` 查看：

```
(dlv) bp
Breakpoint runtime-fatal-throw (enabled) at 0x1038fc0 for runtime.fatalthrow() /usr/local/Cellar/go/1.16.2/libexec/src/runtime/panic.go:1163 (0)
Breakpoint unrecovered-panic (enabled) at 0x1039040 for runtime.fatalpanic() /usr/local/Cellar/go/1.16.2/libexec/src/runtime/panic.go:1190 (0)
	print runtime.curg._panic.arg
Breakpoint 1 (enabled) at 0x10cbab3 for main.main() ./main.go:9 (0)
Breakpoint 2 (enabled) at 0x10cb92c for github.com/eddycjy/awesome-project/stringer.Reverse() ./stringer/string.go:6 (0)
```

查看断点情况后，若有部分已经排除了，可以调用关键字 `clearall` 对一些断点清除：

```
(dlv) clearall main.main
Breakpoint 1 (enabled) cleared at 0x10cbab3 for main.main() ./main.go:9
```

若不指点断点，则会默认清除全部断点。

在日常的 Go 工程中，若都从 main 方法进入就太繁琐了。我们可以直接借助函数名进行调式定位：

```
(dlv) funcs Reverse
github.com/eddycjy/awesome-project/stringer.Reverse
(dlv) b stringer.Reverse
Breakpoint 3 (enabled) set at 0x10cb87b for github.com/eddycjy/awesome-project/stringer.Reverse() ./stringer/string.go:3
(dlv) c
> github.com/eddycjy/awesome-project/stringer.Reverse() ./stringer/string.go:3 (hits goroutine(1):1 total:1) (PC: 0x10cb87b)
     1:	package stringer
     2:	
=>   3:	func Reverse(s string) string {
     4:		r := []rune(s)
     5:		for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
     6:			r[i], r[j] = r[j], r[i]
     7:		}
     8:		return string(r)
```

紧接着其他步骤都与先前的一样，进行具体的调试就好了。我们也可以借助 Go 语言的公共函数进行计算：

```
(dlv) p len(r)-1
6
```

也可以借助关键字 `vars` 查看某个包下的所有全局变量的值，例如：`vars main`。这种方式对于查看全局变量的情况非常有帮助。

排查完毕后，执行关键字 `exit` 就可以愉快的退出了：

```
(dlv) exit
```

解决完问题，可以下班了 ：）

## 总结

在 Go 语言中，Delve 调试工具是与 Go 语言亲和度最高的，因为 Delve 是 Go 语言实现的。其在我们日常工作中，非常常用。

像是假设程序的 for 循环运行到第 N 次才出现 BUG 时，我们就可以通过断点对应的方法和代码块，再设置变量的值，进行具体的查看，就可以解决。

