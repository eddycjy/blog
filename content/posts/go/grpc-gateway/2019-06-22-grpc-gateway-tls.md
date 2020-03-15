---

title:      "「连载四」gRPC+gRPC Gateway 能不能不用证书？"
date:       2019-06-22 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - grpc-gateway
---

如果你以前有涉猎过 gRPC+gRPC Gateway 这两个组件，你肯定会遇到这个问题，就是 **“为什么非得开 TLS，才能够实现同端口双流量，能不能不开？”** 又或是 **“我不想用证书就实现这些功能，行不行？”**。我被无数的人问过无数次这些问题，也说服过很多人，但说服归说服，不代表放弃。前年不行，不代表今年不行，在今天我希望分享来龙去脉和具体的实现方式给你。

![image](https://s2.ax1x.com/2020/02/27/3dLBAx.png)

## 过去

### 为什么 h2 不行

因为 `net/http2` 仅支持 "h2" 标识，而 "h2" 标识 HTTP/2 必须使用传输层安全性（TLS）的协议，此标识符用于 TLS 应用层协议协商字段以及识别 HTTP/2 over TLS。

简单来讲，也就 `net/http2` 必须使用 TLS 来交互。通俗来讲就要用证书，那么理所当然，也就无法支持非 TLS 的情况了。

### 寻找 h2c

那这条路不行，我们再想想别的路？那就是 HTTP/2 规范中的 "h2c" 标识了，"h2c" 标识允许通过明文 TCP 运行 HTTP/2 的协议，此标识符用于 HTTP/1.1 升级标头字段以及标识 HTTP/2 over TCP。

但是这条路，早在 2015 年就已经有在 [issue](https://github.com/golang/go/issues/13128#issuecomment-153193762) 中进行讨论，当时 @bradfitz 明确表示 “不打算支持 h2c，对仅支持 TLS 的情况非常满意，一年后再问我一次”，原文回复如下：

> We do not plan to support h2c. I don't want to receive bug reports from users who get bitten by transparent proxies messing with h2c. Also, until there's widespread browser support, it's not interesting. I am also not interested in being the chicken or the egg to get browser support going. I'm very happy with the TLS-only situation, and things like https://LetsEncrypt.org/ will make TLS much easier (and automatic) soon.

> Ask me again in one year.

### 琢磨其他方式

#### 使用 cmux

基于多路复用器 [soheilhy/cmux](https://github.com/soheilhy/cmux) 的另类实现 [Stoakes/grpc-gateway-example](https://github.com/Stoakes/grpc-gateway-example)。若对 `cmux` 的实现方式感兴趣，还可以看看 [《Golang: Run multiple services on one port》](https://blog.dgraph.io/post/cmux/)。

#### 使用第三方 h2

- [veqryn/h2c](https://github.com/veqryn/h2c)

这种属于自己实现了 h2c 的逻辑，以此达到效果。

## 现在

经过社区的不断讨论，最后在 2018 年 6 月，代表 "h2c" 标志的 `golang.org/x/net/http2/h2c` 标准库正式合并进来，自此我们就可以使用官方标准库（h2c），这个标准库实现了 HTTP/2 的未加密模式，因此我们就可以利用该标准库在同个端口上既提供 HTTP/1.1 又提供 HTTP/2 的功能了。

### 使用标准库 h2c 

```
import (
	...

	"golang.org/x/net/http2"
	"golang.org/x/net/http2/h2c"
	"google.golang.org/grpc"

	"github.com/grpc-ecosystem/grpc-gateway/runtime"

	pb "github.com/EDDYCJY/go-grpc-example/proto"
)

...

func grpcHandlerFunc(grpcServer *grpc.Server, otherHandler http.Handler) http.Handler {
	return h2c.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.ProtoMajor == 2 && strings.Contains(r.Header.Get("Content-Type"), "application/grpc") {
			grpcServer.ServeHTTP(w, r)
		} else {
			otherHandler.ServeHTTP(w, r)
		}
	}), &http2.Server{})
}

func main() {
	server := grpc.NewServer()

	pb.RegisterSearchServiceServer(server, &SearchService{})

	mux := http.NewServeMux()
	gwmux := runtime.NewServeMux()
	dopts := []grpc.DialOption{grpc.WithInsecure()}

	err := pb.RegisterSearchServiceHandlerFromEndpoint(context.Background(), gwmux, "localhost:"+PORT, dopts)
	...
	mux.Handle("/", gwmux)
	http.ListenAndServe(":"+PORT, grpcHandlerFunc(server, mux))
}
```

我们可以看到关键之处在于调用了 `h2c.NewHandler` 方法进行了特殊处理，`h2c.NewHandler` 会返回一个 `http.handler`，主要的内部逻辑是拦截了所有 `h2c` 流量，然后根据不同的请求流量类型将其劫持并重定向到相应的 `Hander` 中去处理。

### 验证

#### HTTP/1.1

```
$ curl -X GET 'http://127.0.0.1:9005/search?request=EDDYCJY'
{"response":"EDDYCJY"}
```

#### HTTP/2(gRPC)

```
...
func main() {
	conn, err := grpc.Dial(":"+PORT, grpc.WithInsecure())
	...
	client := pb.NewSearchServiceClient(conn)
	resp, err := client.Search(context.Background(), &pb.SearchRequest{
		Request: "gRPC",
	})
}
```
输出结果：

```
$ go run main.go
2019/06/21 20:04:09 resp: gRPC h2c Server
```

## 总结

在本文中我介绍了大致的前因后果，且介绍了几种解决方法，我建议你选择官方的 `h2c` 标准库去实现这个功能，也简单。在最后，不管你是否曾经为这个问题烦恼过许久，又或者正在纠结，都希望这篇文章能够帮到你。

## 参考

- https://github.com/golang/go/issues/13128
- https://github.com/golang/go/issues/14141
- https://github.com/golang/net/commit/c4299a1a0d8524c11563db160fbf9bddbceadb21
- https://go-review.googlesource.com/c/net/+/112997/
