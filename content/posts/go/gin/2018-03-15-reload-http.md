---

title:      "「连载七」优雅的重启服务"
date:       2018-03-15 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - gin
---

## 知识点

- 信号量的了解。
- 应用热更新。

## 本文目标

在前面编写案例代码时，我相信你会想到，每次更新完代码，更新完配置文件后，就直接这么 `ctrl+c` 真的没问题吗，`ctrl+c`到底做了些什么事情呢？

在这一节中我们简单讲述 `ctrl+c` 背后的**信号**以及如何在`Gin`中**优雅的重启服务**，也就是对 `HTTP` 服务进行热更新。

## ctrl + c

> 内核在某些情况下发送信号，比如在进程往一个已经关闭的管道写数据时会产生`SIGPIPE`信号

在终端执行特定的组合键可以使系统发送特定的信号给此进程，完成一系列的动作

| 命令      | 信号    | 含义                                                                                                    |
| --------- | ------- | ------------------------------------------------------------------------------------------------------- |
| ctrl + c  | SIGINT  | 强制进程结束                                                                                            |
| ctrl + z  | SIGTSTP | 任务中断，进程挂起                                                                                      |
| ctrl + \  | SIGQUIT | 进程结束 和 `dump core`                                                                                 |
| ctrl + d  |         | EOF                                                                                                     |
|           | SIGHUP  | 终止收到该信号的进程。若程序中没有捕捉该信号，当收到该信号时，进程就会退出（常用于 重启、重新加载进程） |

因此在我们执行`ctrl + c`关闭`gin`服务端时，**会强制进程结束，导致正在访问的用户等出现问题**

常见的 `kill -9 pid` 会发送 `SIGKILL` 信号给进程，也是类似的结果

### 信号

本段中反复出现**信号**是什么呢？

信号是 `Unix` 、类 `Unix` 以及其他 `POSIX` 兼容的操作系统中进程间通讯的一种有限制的方式

它是一种异步的通知机制，用来提醒进程一个事件（硬件异常、程序执行异常、外部发出信号）已经发生。当一个信号发送给一个进程，操作系统中断了进程正常的控制流程。此时，任何非原子操作都将被中断。如果进程定义了信号的处理函数，那么它将被执行，否则就执行默认的处理函数

### 所有信号

```
$ kill -l
 1) SIGHUP   2) SIGINT   3) SIGQUIT  4) SIGILL   5) SIGTRAP
 6) SIGABRT  7) SIGBUS   8) SIGFPE   9) SIGKILL 10) SIGUSR1
11) SIGSEGV 12) SIGUSR2 13) SIGPIPE 14) SIGALRM 15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD 18) SIGCONT 19) SIGSTOP 20) SIGTSTP
21) SIGTTIN 22) SIGTTOU 23) SIGURG  24) SIGXCPU 25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF 28) SIGWINCH    29) SIGIO   30) SIGPWR
31) SIGSYS  34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

## 怎样算优雅

### 目的

- 不关闭现有连接（正在运行中的程序）
- 新的进程启动并替代旧进程
- 新的进程接管新的连接
- 连接要随时响应用户的请求，当用户仍在请求旧进程时要保持连接，新用户应请求新进程，不可以出现拒绝请求的情况

### 流程

1、替换可执行文件或修改配置文件

2、发送信号量 `SIGHUP`

3、拒绝新连接请求旧进程，但要保证已有连接正常

4、启动新的子进程

5、新的子进程开始 `Accet`

6、系统将新的请求转交新的子进程

7、旧进程处理完所有旧连接后正常结束

## 实现优雅重启

### endless

> Zero downtime restarts for golang HTTP and HTTPS servers. (for golang 1.3+)

我们借助 [fvbock/endless](https://github.com/fvbock/endless) 来实现 `Golang HTTP/HTTPS` 服务重新启动的零停机

`endless server` 监听以下几种信号量：

- syscall.SIGHUP：触发 `fork` 子进程和重新启动
- syscall.SIGUSR1/syscall.SIGTSTP：被监听，但不会触发任何动作
- syscall.SIGUSR2：触发 `hammerTime`
- syscall.SIGINT/syscall.SIGTERM：触发服务器关闭（会完成正在运行的请求）

`endless` 正正是依靠监听这些**信号量**，完成管控的一系列动作

#### 安装

```
go get -u github.com/fvbock/endless
```

#### 编写

打开 [gin-blog](https://github.com/EDDYCJY/go-gin-example) 的 `main.go`文件，修改文件：

```go
package main

import (
    "fmt"
    "log"
    "syscall"

    "github.com/fvbock/endless"

    "gin-blog/routers"
    "gin-blog/pkg/setting"
)

func main() {
    endless.DefaultReadTimeOut = setting.ReadTimeout
    endless.DefaultWriteTimeOut = setting.WriteTimeout
    endless.DefaultMaxHeaderBytes = 1 << 20
    endPoint := fmt.Sprintf(":%d", setting.HTTPPort)

    server := endless.NewServer(endPoint, routers.InitRouter())
    server.BeforeBegin = func(add string) {
        log.Printf("Actual pid is %d", syscall.Getpid())
    }

    err := server.ListenAndServe()
    if err != nil {
        log.Printf("Server err: %v", err)
    }
}
```

`endless.NewServer` 返回一个初始化的 `endlessServer` 对象，在 `BeforeBegin` 时输出当前进程的 `pid`，调用 `ListenAndServe` 将实际“启动”服务

#### 验证

##### **编译**

```
$ go build main.go
```

##### **执行**

```
$ ./main
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
...
Actual pid is 48601
```

启动成功后，输出了`pid`为 48601；在另外一个终端执行 `kill -1 48601` ，检验先前服务的终端效果

```
[root@localhost go-gin-example]# ./main
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /auth                     --> ...
[GIN-debug] GET    /api/v1/tags              --> ...
...

Actual pid is 48601

...

Actual pid is 48755
48601 Received SIGTERM.
48601 [::]:8000 Listener closed.
48601 Waiting for connections to finish...
48601 Serve() returning...
Server err: accept tcp [::]:8000: use of closed network connection
```

可以看到该命令已经挂起，并且 `fork` 了新的子进程 `pid` 为 `48755`

```
48601 Received SIGTERM.
48601 [::]:8000 Listener closed.
48601 Waiting for connections to finish...
48601 Serve() returning...
Server err: accept tcp [::]:8000: use of closed network connection
```

大致意思为主进程（`pid`为 48601）接受到 `SIGTERM` 信号量，关闭主进程的监听并且等待正在执行的请求完成；这与我们先前的描述一致

##### **唤醒**

这时候在 `postman` 上再次访问我们的接口，你可以惊喜的发现，他“复活”了！

```
Actual pid is 48755
48601 Received SIGTERM.
48601 [::]:8000 Listener closed.
48601 Waiting for connections to finish...
48601 Serve() returning...
Server err: accept tcp [::]:8000: use of closed network connection


$ [GIN] 2018/03/15 - 13:00:16 | 200 |     188.096µs |   192.168.111.1 | GET      /api/v1/tags...

```

这就完成了一次正向的流转了

你想想，每次更新发布、或者修改配置文件等，只需要给该进程发送**SIGTERM 信号**，而不需要强制结束应用，是多么便捷又安全的事！

#### 问题

`endless` 热更新是采取创建子进程后，将原进程退出的方式，这点不符合守护进程的要求

### http.Server - Shutdown()

如果你的`Golang >= 1.8`，也可以考虑使用 `http.Server` 的 [Shutdown](https://golang.org/pkg/net/http/#Server.Shutdown) 方法

```go
package main

import (
    "fmt"
    "net/http"
    "context"
    "log"
    "os"
    "os/signal"
    "time"


    "gin-blog/routers"
    "gin-blog/pkg/setting"
)

func main() {
    router := routers.InitRouter()

    s := &http.Server{
        Addr:           fmt.Sprintf(":%d", setting.HTTPPort),
        Handler:        router,
        ReadTimeout:    setting.ReadTimeout,
        WriteTimeout:   setting.WriteTimeout,
        MaxHeaderBytes: 1 << 20,
    }

    go func() {
        if err := s.ListenAndServe(); err != nil {
            log.Printf("Listen: %s\n", err)
        }
    }()

    quit := make(chan os.Signal)
    signal.Notify(quit, os.Interrupt)
    <- quit

    log.Println("Shutdown Server ...")

    ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
    defer cancel()
    if err := s.Shutdown(ctx); err != nil {
        log.Fatal("Server Shutdown:", err)
    }

    log.Println("Server exiting")
}
```

## 小结

在日常的服务中，优雅的重启（热更新）是非常重要的一环。而 `Golang` 在 `HTTP` 服务方面的热更新也有不少方案了，我们应该根据实际应用场景挑选最合适的

## 参考

### 本系列示例代码

- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

### 拓展阅读

- [manners](https://github.com/braintree/manners)
- [graceful](https://github.com/tylerb/graceful)
- [grace](https://github.com/facebookgo/grace)
- [plugin: new package for loading plugins · golang/go@0cbb12f · GitHub](https://github.com/golang/go/commit/0cbb12f0bbaeb3893b3d011fdb1a270291747ab0)

## 关于

### 修改记录

- 第一版：2018 年 02 月 16 日发布文章
- 第二版：2019 年 10 月 01 日修改文章

## ？

如果有任何疑问或错误，欢迎在 [issues](https://github.com/EDDYCJY/blog) 进行提问或给予修正意见，如果喜欢或对你有所帮助，欢迎 Star，对作者是一种鼓励和推进。

### 我的公众号

![image](https://image.eddycjy.com/8d0b0c3a11e74efd5fdfd7910257e70b.jpg)