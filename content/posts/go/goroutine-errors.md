---
title: "多 Goroutine 如何优雅处理错误？"
date: 2021-12-31T12:54:50+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

在 Go 语言中，goroutine 的使用是非常频繁的，因此在日常编码的时候我们会遇到一个问题，那就是 goroutine 里面的错误处理，怎么做比较好？

![](https://files.mdnice.com/user/3610/72758b15-f9b7-4437-ba17-b37a36f285ae.png)


这是来自我读者群的问题。作为一个宠粉煎鱼，我默默记下了这个技术话题。今天煎鱼就大家来看看多 goroutine 的错误处理机制也有哪些！

一般来讲，我们的业务代码会是：

```golang
func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		log.Println("脑子进煎鱼了")
		wg.Done()
	}()
	go func() {
		log.Println("煎鱼想报错...")
		wg.Done()
	}()

	time.Sleep(time.Second)
}
```

在上述代码中，我们运行了多个 goroutine。但我想抛出 error 的错误信息出来，似乎没什么好办法...

## 通过错误日志记录

为此，业务代码中常见的第一种方法：通过把错误记录写入日志文件中，再结合相关的 logtail 进行采集和梳理。

但这又会引入新的问题，那就是调用错误日志的方法写的到处都是。代码结构也比较乱，不直观。

最重要的是无法针对 error 做特定的逻辑处理和流转。

## 利用 channel 传输

这时候大家可能会想到 Go 的经典哲学：**不要通过共享内存来通信，而是通过通信来实现内存共享**（Do not communicate by sharing memory; instead, share memory by communicating）。

第二种的方法：利用 channel 来传输多个 goroutine 中的 errors：

```golang
func main() {
	gerrors := make(chan error)
	wgDone := make(chan bool)

	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		wg.Done()
	}()
	go func() {
		err := returnError()
		if err != nil {
			gerrors <- err
		}
		wg.Done()
	}()

	go func() {
		wg.Wait()
		close(wgDone)
	}()

	select {
	case <-wgDone:
		break
	case err := <-gerrors:
		close(gerrors)
		fmt.Println(err)
	}

	time.Sleep(time.Second)
}

func returnError() error {
	return errors.New("煎鱼报错了...")
}
```

输出结果：

```golang
煎鱼报错了...
```

虽然使用 channel 后已经方便了不少。但自己编写 channel 总是需要关心一些非业务向的逻辑。

## 借助 sync/errgroup

因此第三种方法，就是使用官方提供的 `sync/errgroup` 标准库：

```golang
type Group
    func WithContext(ctx context.Context) (*Group, context.Context)
    func (g *Group) Go(f func() error)
    func (g *Group) Wait() error
```

- Go：启动一个协程，在新的 goroutine 中调用给定的函数。
- Wait：等待协程结束，直到来自 Go 方法的所有函数调用都返回，然后返回其中的第一个非零错误（如果有的话）。

结合其特性能够非常便捷的针对多 goroutine 进行错误处理：

```golang
func main() {
	g := new(errgroup.Group)
	var urls = []string{
		"http://www.golang.org/",
		"https://golang2.eddycjy.com/",
		"https://eddycjy.com/",
	}
	for _, url := range urls {
		url := url
		g.Go(func() error {
			resp, err := http.Get(url)
			if err == nil {
				resp.Body.Close()
			}
			return err
		})
	}
	if err := g.Wait(); err == nil {
		fmt.Println("Successfully fetched all URLs.")
	} else {
		fmt.Printf("Errors: %+v", err)
	}
}
```

在上述代码中，其表现的是爬虫的案例。每一个计划新起的 goroutine 都直接使用 `Group.Go` 方法。在等待和错误上，直接调用 `Group.Wait` 方法就可以了。

使用标准库 `sync/errgroup` 这种方法的好处就是不需要关注非业务逻辑的控制代码，比较省心省力。

## 进阶使用

在真实的工程代码中，我们还可以基于 `sync/errgroup` 实现一个 http server 的启动和关闭 ，以及 linux signal 信号的注册和处理。以此保证能够实现一个 http server 退出，全部注销退出。

参考代码（@via 毛老师）如下：

```golang
func main() {
	g, ctx := errgroup.WithContext(context.Background())
	svr := http.NewServer()
	// http server
	g.Go(func() error {
		fmt.Println("http")
		go func() {
			<-ctx.Done()
			fmt.Println("http ctx done")
			svr.Shutdown(context.TODO())
		}()
		return svr.Start()
	})

	// signal
	g.Go(func() error {
		exitSignals := []os.Signal{os.Interrupt, syscall.SIGTERM, syscall.SIGQUIT, syscall.SIGINT} // SIGTERM is POSIX specific
		sig := make(chan os.Signal, len(exitSignals))
		signal.Notify(sig, exitSignals...)
		for {
			fmt.Println("signal")
			select {
			case <-ctx.Done():
				fmt.Println("signal ctx done")
				return ctx.Err()
			case <-sig:
				// do something
				return nil
			}
		}
	})

	// inject error
	g.Go(func() error {
		fmt.Println("inject")
		time.Sleep(time.Second)
		fmt.Println("inject finish")
		return errors.New("inject error")
	})

	err := g.Wait() // first error return
	fmt.Println(err)
}
```

内部基础框架有非常有这种代码，有兴趣的可以自己模仿着写一遍，收货会很多。

## 总结

在 Go 语言中 goroutine 是非常常用的一种方法，为此我们需要更了解 goroutine 配套的上下游（像是 context、error 处理等），应该如何用什么来保证。

再在团队中形成一定的共识和规范，这么工程代码阅读起来就会比较的舒适，一些很坑的隐藏 BUG 也会少很多 ：）