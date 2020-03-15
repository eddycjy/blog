---

title:      "聊一聊，Go 的相对路径问题"
date:       2018-03-13 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
---

## 前言

`Golang` 中存在各种运行方式，如何**正确的引用文件路径**成为一个值得商议的问题

以 [gin-blog](https://github.com/EDDYCJY/go-gin-example) 为例，当我们在项目根目录下，执行 `go run main.go` 时能够正常运行（`go build`也是正常的）
```
[$ gin-blog]# go run main.go
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:    export GIN_MODE=release
 - using code:    gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /api/v1/tags              --> gin-blog/routers/api/v1.GetTags (3 handlers)
...
```


那么在不同的目录层级下，不同的方式运行，又是怎么样的呢，带着我们的疑问去学习

## 问题
1、 go run
我们上移目录层级，到 `$GOPATH/src` 下，执行 `go run gin-blog/main.go`
```
[$ src]# go run gin-blog/main.go
2018/03/12 16:06:13 Fail to parse 'conf/app.ini': open conf/app.ini: no such file or directory
exit status 1
```

2、 go build，执行 `./gin-blog/main`
```
[$ src]# ./gin-blog/main
2018/03/12 16:49:35 Fail to parse 'conf/app.ini': open conf/app.ini: no such file or directory
```

这时候你要打一个大大的问号，就是我的程序读取到什么地方去了

--- 

我们通过分析得知，**`Golang`的相对路径是相对于执行命令时的目录**；自然也就读取不到了


## 思考

既然已经知道问题的所在点，我们就可以寻思做点什么 : )

我们想到相对路径是相对执行命令的目录，那么我们获取可执行文件的地址，拼接起来不就好了吗？

## 实践

我们编写**获取当前可执行文件路径的方法**
```
import (
	"path/filepath"
	"os"
	"os/exec"
	"string"
)

func GetAppPath() string {
    file, _ := exec.LookPath(os.Args[0])
    path, _ := filepath.Abs(file)
    index := strings.LastIndex(path, string(os.PathSeparator))

    return path[:index]
}
```
将其放到启动代码处查看路径
```
log.Println(GetAppPath())
```

我们分别执行以下两个命令，查看输出结果
1、 go run 
```
$ go run main.go
2018/03/12 18:45:40 /tmp/go-build962610262/b001/exe
```
2、 go build
```
$ ./main
2018/03/12 18:49:44 $GOPATH/src/gin-blog

```

## 剖析

我们聚焦在 `go run` 的输出结果上，发现它是一个临时文件的地址，这是为什么呢？

在`go help run`中，我们可以看到
```
Run compiles and runs the main package comprising the named Go source files.
A Go source file is defined to be a file ending in a literal ".go" suffix.
```

也就是 `go run` 执行时会将文件放到 `/tmp/go-build...` 目录下，编译并运行

因此`go run main.go`出现`/tmp/go-build962610262/b001/exe`结果也不奇怪了，因为它已经跑到临时目录下去执行可执行文件了

---

这就已经很清楚了，那么我们想想，会出现哪些问题呢

- 依赖相对路径的文件，出现路径出错的问题
- `go run` 和 `go build` 不一样，一个到临时目录下执行，一个可手动在编译后的目录下执行，路径的处理方式会不同
- 不断`go run`，不断产生新的临时文件


这其实就是**根本原因**了，因为 `go run` 和 `go build` 的编译文件执行路径并不同，执行的层级也有可能不一样，自然而然就出现各种读取不到的奇怪问题了


## 解决方案

**一、获取编译后的可执行文件路径**

1、 将配置文件的相对路径与`GetAppPath()`的结果相拼接，可解决`go build main.go`的可执行文件跨目录执行的问题（如：`./src/gin-blog/main`）
```
import (
	"path/filepath"
	"os"
	"os/exec"
	"string"
)

func GetAppPath() string {
    file, _ := exec.LookPath(os.Args[0])
    path, _ := filepath.Abs(file)
    index := strings.LastIndex(path, string(os.PathSeparator))

    return path[:index]
}
```

但是这种方式，对于`go run`依旧无效，这时候就需要2来补救

2、 通过传递参数指定路径，可解决`go run`的问题
```
package main

import (
    "flag"
    "fmt"
)

func main() {
    var appPath string
    flag.StringVar(&appPath, "app-path", "app-path")
    flag.Parse()
    fmt.Printf("App path: %s", appPath)
}
```
运行
```
go run main.go --app-path "Your project address"
```

**二、增加`os.Getwd()`进行多层判断**

参见 [beego](https://github.com/astaxie/beego/blob/master/config.go#L133-L146) 读取 `app.conf` 的代码

该写法可兼容 `go build` 和在项目根目录执行 `go run` ，但是若跨目录执行 `go run` 就不行


**三、配置全局系统变量**

我们可以通过`os.Getenv`来获取系统全局变量，然后与相对路径进行拼接

1、 设置项目工作区

简单来说，就是设置项目（应用）的工作路径，然后与配置文件、日志文件等相对路径进行拼接，达到相对的绝对路径来保证路径一致

参见 [gogs](https://github.com/gogits/gogs/blob/master/pkg/setting/setting.go#L351) 读取`GOGS_WORK_DIR`进行拼接的代码

2、 利用系统自带变量

简单来说就是通过系统自带的全局变量，例如`$HOME`等，将配置文件存放在`$HOME/conf`或`/etc/conf`下

这样子就能更加固定的存放配置文件，**不需要额外去设置一个环境变量**

（这点今早与一位SFer讨论了一波，感谢）

## 拓展

`go test` 在一些场景下也会遇到路径问题，因为`go test`只能够在当前目录执行，所以在执行测试用例的时候，你的执行目录已经是测试目录了

需要注意的是，如果采用获取外部参数的办法，用 `os.args` 时，`go test -args` 和 `go run`、`go build` 会有命令行参数位置的不一致问题

## 小结

这三种解决方案，在目前可见的开源项目或介绍中都能找到这些的身影

优缺点也是显而易见的，我认为应在**不同项目选定合适的解决方案**即可

建议大家不要强依赖读取配置文件的模块，应当将其“堆积木”化，**需要什么配置才去注册什么配置变量**，可以解决一部分的问题

大家又有什么想法呢，一起讨论一波？

