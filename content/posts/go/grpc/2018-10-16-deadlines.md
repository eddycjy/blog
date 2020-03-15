---

title:      "「连载九」gRPC Deadlines"
date:       2018-10-16 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - grpc
---

## 前言

在前面的章节中，已经介绍了 gRPC 的基本用法。那你想想，让它这么裸跑真的没问题吗？

那么，肯定是有问题了。今天将介绍 gRPC Deadlines 的用法，这一个必备技巧。内容也比较简单

## Deadlines

Deadlines 意指截止时间，在 gRPC 中强调 TL;DR（Too long, Don't read）并建议**始终设定截止日期**，为什么呢？

### 为什么要设置

当未设置 Deadlines 时，将采用默认的 DEADLINE_EXCEEDED（这个时间非常大）

如果产生了阻塞等待，就会造成大量正在进行的请求都会被保留，并且所有请求都有可能达到最大超时

这会使服务面临资源耗尽的风险，例如内存，这会增加服务的延迟，或者在最坏的情况下可能导致整个进程崩溃

## gRPC

### Client

```
func main() {
    ...
	ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(time.Duration(5 * time.Second)))
	defer cancel()

	client := pb.NewSearchServiceClient(conn)
	resp, err := client.Search(ctx, &pb.SearchRequest{
		Request: "gRPC",
	})
	if err != nil {
		statusErr, ok := status.FromError(err)
		if ok {
			if statusErr.Code() == codes.DeadlineExceeded {
				log.Fatalln("client.Search err: deadline")
			}
		}

		log.Fatalf("client.Search err: %v", err)
	}

	log.Printf("resp: %s", resp.GetResponse())
}
```

- context.WithDeadline：会返回最终上下文截止时间。第一个形参为父上下文，第二个形参为调整的截止时间。若父级时间早于子级时间，则以父级时间为准，否则以子级时间为最终截止时间

```
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(true, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

- context.WithTimeout：很常见的另外一个方法，是便捷操作。实际上是对于 WithDeadline 的封装

```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

- status.FromError：返回 GRPCStatus 的具体错误码，若为非法，则直接返回 `codes.Unknown`

### Server

```
type SearchService struct{}

func (s *SearchService) Search(ctx context.Context, r *pb.SearchRequest) (*pb.SearchResponse, error) {
	for i := 0; i < 5; i++  {
		if ctx.Err() == context.Canceled {
			return nil, status.Errorf(codes.Canceled, "SearchService.Search canceled")
		}

		time.Sleep(1 * time.Second)
	}

	return &pb.SearchResponse{Response: r.GetRequest() + " Server"}, nil
}

func main() {
	...
}
```

而在 Server 端，由于 Client 已经设置了截止时间。Server 势必要去检测它

否则如果 Client 已经结束掉了，Server 还傻傻的在那执行，这对资源是一种极大的浪费

因此在这里需要用 `ctx.Err() == context.Canceled` 进行判断，为了模拟场景我们加了循环和睡眠 🤔

### 验证

重新启动 server.go 和 client.go，得到结果：

```
$ go run client.go
2018/10/06 17:45:55 client.Search err: deadline
exit status 1
```

## 总结

本章节比较简单，你需要知道以下知识点：

- 怎么设置 Deadlines
- 为什么要设置 Deadlines

你要清楚地明白到，gRPC Deadlines 是很重要的，否则这小小的功能点就会要了你生产的命 🤫

## 参考

### 本系列示例代码

- [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)

### 资料

- [gRPC and Deadlines](https://grpc.io/blog/deadlines)
