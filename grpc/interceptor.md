# 4.6 Unary and Stream interceptor

项目地址：https://github.com/EDDYCJY/go-grpc-example

## 前言

我想在每个 RPC 方法的前或后做某些事情，怎么做？

本章节将要介绍的拦截器（interceptor），就能帮你在合适的地方实现这些功能 🤫

## 有几种方法

在 gRPC 中，大类可分为两种 RPC 方法，与拦截器的对应关系是：

- 普通方法：一元拦截器（grpc.UnaryInterceptor）
- 流方法：流拦截器（grpc.StreamInterceptor）


## 看一看

### grpc.UnaryInterceptor

```
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {
	return func(o *options) {
		if o.unaryInt != nil {
			panic("The unary server interceptor was already set and may not be reset.")
		}
		o.unaryInt = i
	}
}
```

```
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

通过查看源码可得知，要完成一个拦截器需要实现 `UnaryServerInterceptor` 方法。形参如下：

- ctx context.Context：请求上下文
- req interface{}：RPC 方法的请求参数
- info *UnaryServerInfo：RPC 方法的所有信息
- handler UnaryHandler：RPC 方法本身

### grpc.StreamInterceptor

```
func StreamInterceptor(i StreamServerInterceptor) ServerOption
```

```
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

StreamServerInterceptor 与 UnaryServerInterceptor 形参的意义是一样，不再赘述

### 如何实现多个拦截器

另外，可以发现 gRPC 本身居然只能设置一个拦截器，难道所有的逻辑都只能写在一起？

关于这一点，你可以放心。采用开源项目 [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware) 就可以解决这个问题，本章也会使用它 😄

```
import "github.com/grpc-ecosystem/go-grpc-middleware"

myServer := grpc.NewServer(
    grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
        ...
    )),
    grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
       ...
    )),
)
```

## gRPC

从本节开始编写 gRPC interceptor 的代码，我们会将实现以下拦截器：

- logging：RPC 方法的入参出参的日志输出
- recover：RPC 方法的异常保护和日志输出

### 实现 interceptor

#### logging

```
func LoggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	log.Printf("gRPC method: %s, %v", info.FullMethod, req)
	resp, err := handler(ctx, req)
	log.Printf("gRPC method: %s, %v", info.FullMethod, resp)
	return resp, err
}
```

#### recover

```
func RecoveryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
	defer func() {
		if e := recover(); e != nil {
			debug.PrintStack()
			err = status.Errorf(codes.Internal, "Panic err: %v", e)
		}
	}()

	return handler(ctx, req)
}
```

### Server

```
import (
	"context"
	"crypto/tls"
	"crypto/x509"
	"errors"
	"io/ioutil"
	"log"
	"net"
	"runtime/debug"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/status"
	"google.golang.org/grpc/codes"
	"github.com/grpc-ecosystem/go-grpc-middleware"

	pb "github.com/EDDYCJY/go-grpc-example/proto"
)

...

func main() {
	c, err := GetTLSCredentialsByCA()
	if err != nil {
		log.Fatalf("GetTLSCredentialsByCA err: %v", err)
	}

	opts := []grpc.ServerOption{
		grpc.Creds(c),
		grpc_middleware.WithUnaryServerChain(
			RecoveryInterceptor,
			LoggingInterceptor,
		),
	}

	server := grpc.NewServer(opts...)
	pb.RegisterSearchServiceServer(server, &SearchService{})

	lis, err := net.Listen("tcp", ":"+PORT)
	if err != nil {
		log.Fatalf("net.Listen err: %v", err)
	}

	server.Serve(lis)
}
```

## 验证

### logging

启动 simple_server/server.go，执行 simple_client/client.go 发起请求，得到结果：

```
$ go run server.go
2018/10/02 13:46:35 gRPC method: /proto.SearchService/Search, request:"gRPC" 
2018/10/02 13:46:35 gRPC method: /proto.SearchService/Search, response:"gRPC Server"
```

### recover

在 RPC 方法中人为地制造运行时错误，再重复启动 server/client.go，得到结果：

#### client

```
$ go run client.go
2018/10/02 13:19:03 client.Search err: rpc error: code = Internal desc = Panic err: assignment to entry in nil map
exit status 1
```

#### server

```
$ go run server.go
goroutine 23 [running]:
runtime/debug.Stack(0xc420223588, 0x1033da9, 0xc420001980)
	/usr/local/Cellar/go/1.10.1/libexec/src/runtime/debug/stack.go:24 +0xa7
runtime/debug.PrintStack()
	/usr/local/Cellar/go/1.10.1/libexec/src/runtime/debug/stack.go:16 +0x22
main.RecoveryInterceptor.func1(0xc420223a10)
...
```

检查服务是否仍然运行，即可知道 Recovery 是否成功生效

## 总结

通过本章节，你可以学会最常见的拦截器使用方法。接下来其它“新”需求只要举一反三即可 😈

## 参考
### 本系列示例代码
- [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)