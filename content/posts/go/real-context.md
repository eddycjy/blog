---
title: "分享 Go 使用 Context 的正式姿势"
date: 2021-12-31T12:54:54+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

在 Go 语言中，Goroutine（协程），也就是关键字 `go` 是一个家喻户晓的高级用法。这起的非常妙，说到 Go，就会想到这一门语言，想到 goroutine 这一关键字，而与之关联最深的就是 context。

![](https://files.mdnice.com/user/3610/2f26bd64-e9bd-4c89-a9a0-326b8c2b9d00.png)

## 背景

平时在 Go 工程中开发中，几乎所有服务端（例如：HTTP Server）的默认实现，都在处理请求时新起 goroutine 进行处理。

但一开始存在一个问题，那就是当一个请求被取消或超时时，所有在该请求上工作的 goroutines 应该迅速退出，以便系统可以回收他们正在使用的任何资源。

当年可没有 context 标准库。很折腾。因此 Go 官方在 2014 年正式宣发了 context 标准库，形成一个完整的闭环。

但有了 context 标准库，Go 爱好者们又奇怪了，前段时间我就被问到了：“Go context 的正确使用姿势是怎么样的”？

（一张忘记在哪里被问的隐形截图）

今天这篇文章就由煎鱼带你看看。

## Context 用法

在 Go context 用法中，我们常常将其与 select 关键字结合使用，用于监听其是否结束、取消等。

代码如下：

```golang
const shortDuration = 1 * time.Millisecond

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), shortDuration)
	defer cancel()

	select {
	case <-time.After(1 * time.Second):
		fmt.Println("脑子进煎鱼了")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

输出结果：

```
context deadline exceeded
```

如果是更进一步结合 goroutine 的话，常见的例子是：

```golang
	func(ctx context.Context) <-chan int {
		dst := make(chan int)
		n := 1
		go func() {
			for {
				select {
				case <-ctx.Done():
					return
				case dst <- n:
					n++
				}
			}
		}()
		return dst
	}
```

我们平时工程中会起很多的 goroutine，这时候会在 goroutine 内结合 for+select，针对 context 的事件进行处理，达到跨 goroutine 控制的目的。

## 正确的使用姿势

### 对第三方调用要传入 context

在 Go 语言中，Context 的默认支持已经是约定俗称的规范了。因此在我们对第三方有调用诉求的时候，要传入 context：

```golang
func main() {
	req, err := http.NewRequest("GET", "https://eddycjy.com/", nil)
	if err != nil {
		fmt.Printf("http.NewRequest err: %+v", err)
		return
	}

	ctx, cancel := context.WithTimeout(req.Context(), 50*time.Millisecond)
	defer cancel()

	req = req.WithContext(ctx)
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		fmt.Printf("http.DefaultClient.Do err: %+v", err)
		return
	}
	defer resp.Body.Close()
}
```

这样子由于第三方开源库已经实现了根据 context 的超时控制，那么当你所传入的时间到达时，将会中断调用。

若你发现第三方开源库没支持 context，那建议赶紧跑，换一个。免得在微服务体系下出现级联故障，还没有简单的手段控制，那就很麻烦了。

### 不要将上下文存储在结构类型中

大家会发现，在 Go 语言中，所有的第三方开源库，业务代码。清一色的都会将 context 放在方法的一个入参参数，作为首位形参。

例如：

![](https://files.mdnice.com/user/3610/f79303b3-2070-4d2b-ac6b-840c413d03fd.png)

标准要求：每个方法的第一个参数都将 context 作为第一个参数，并使用 ctx 变量名惯用语。

当然，我们也不能一杆子打死所有情况。确实存在极少数是把 context 放在结构体中的。基本常见于：
- 底层基础库。
- DDD 结构。

每个请求都是独立的，context 自然每个都不一样，想清楚自己的应用使用场景很重要，否则遵循 Go 基本规范就好。


在真实案例来看，有的 Leader 会单纯为了不想频繁传 context 而设计成结构体，结果导致一线 RD 就得天天 NewXXX，甚至有时候忘记了，还得背个小锅。

### 函数调用链必须传播上下文

我们会把 context 作为方法首位，本质目的是为了传播 context，自行完整调用链路上的各类控制：

```golang
func List(ctx context.Context, db *sqlx.DB) ([]User, error) {
	ctx, span := trace.StartSpan(ctx, "internal.user.List")
	defer span.End()

	users := []User{}
	const q = `SELECT * FROM users`

	if err := db.SelectContext(ctx, &users, q); err != nil {
		return nil, errors.Wrap(err, "selecting users")
	}

	return users, nil
}
```

像在上述例子中，我们会把所传入方法的 context 一层层的传进去下一级方法。这里就是将外部的 context 传入 List 方法，再传入 SQL 执行的方法，解决了 SQL 执行语句的时间问题。

### context 的继承和派生

在 Go 标准库 context 中具有以下派生 context 的标准方法：

```golang
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

代码例子如下：

```golang
func handle(w http.ResponseWriter, req *http.Request) {
  // parent context
	timeout, _ := time.ParseDuration(req.FormValue("timeout"))
	ctx, cancel := context.WithTimeout(context.Background(), timeout)

  // chidren context
	newCtx, cancel := context.WithCancel(ctx)
	defer cancel()
	// do something...
}
```

一般会有父级 context 和子级 context 的区别，我们要保证在程序的行为中上下文对于多个 goroutine 同时使用是安全的。并且存在父子级别关系，父级 context 关闭或超时，可以继而影响到子级 context 的程序。

### 不传递 nil context

很多时候我们在创建 context 时，还不知道其具体的作用和下一步用途是什么。

这种时候大家可能会直接使用 `context.Background` 方法：

```golang
var (
   background = new(emptyCtx)
   todo       = new(emptyCtx)
)

func Background() Context {
   return background
}

func TODO() Context {
   return todo
}
```

但在实际的 context 建议中，我们会建议使用 `context.TODO` 方法来创建顶级的 context，直到弄清楚实际 Context 的下一步用途，再进行变更。

### context 仅传递必要的值

我们在使用 context 作为上下文时，经常有信息传递的诉求。像是在 gRPC 中就会有 metadata 的概念，而在 gin 中就会自己封装 context 作为参数管理。

Go 标准库 context 也有提供相关的方法：

```golang
type Context
    func WithValue(parent Context, key, val interface{}) Context
```

代码例子如下：

```golang
func main() {
	type favContextKey string
	f := func(ctx context.Context, k favContextKey) {
		if v := ctx.Value(k); v != nil {
			fmt.Println("found value:", v)
			return
		}
		fmt.Println("key not found:", k)
	}

	k := favContextKey("脑子进")
	ctx := context.WithValue(context.Background(), k, "煎鱼")

	f(ctx, k)
	f(ctx, favContextKey("小咸鱼"))
}
```

输出结果：

```
found value: 煎鱼
key not found: 小咸鱼
```

在规范中，我们建议 context 在传递时，仅携带必要的参数给予其他的方法，或是 goroutine。甚至在 gRPC 中会做严格的出、入上下文参数的控制。

在业务场景上，context 传值适用于传必要的业务核心属性，例如：租户号、小程序ID 等。不要将可选参数放到 context 中，否则可能会一团糟。

## 总结

- 对第三方调用要传入 context，用于控制远程调用。
- 不要将上下文存储在结构类型中，尽可能的作为函数第一位形参传入。
- 函数调用链必须传播上下文，实现完整链路上的控制。
- context 的继承和派生，保证父、子级 context 的联动。
- 不传递 nil context，不确定的 context 应当使用 TODO。
- context 仅传递必要的值，不要让可选参数揉在一起。