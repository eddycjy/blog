# 带入gRPC：对 RPC 方法做自定义认证

项目地址：https://github.com/EDDYCJY/go-grpc-example

## 前言

在前面的章节中，我们介绍了两种（证书算一种）可全局认证的方法：

1. [TLS 证书认证](https://github.com/EDDYCJY/blog/blob/master/golang/gRPC/2018-10-01-%E5%B8%A6%E5%85%A5gRPC-TLS-%E8%AF%81%E4%B9%A6%E8%AE%A4%E8%AF%81.md)
2. [基于 CA 的 TLS 证书认证](https://github.com/EDDYCJY/blog/blob/master/golang/gRPC/2018-10-01-%E5%B8%A6%E5%85%A5gRPC-%E5%9F%BA%E4%BA%8ECA%E7%9A%84TLS%E8%AF%81%E4%B9%A6%E8%AE%A4%E8%AF%81.md)
3. [Unary and Stream interceptor](https://github.com/EDDYCJY/blog/blob/master/golang/gRPC/2018-10-02-%E5%B8%A6%E5%85%A5gRPC-Unary-and-Stream-interceptor.md)


而在实际需求中，常常会对某些模块的 RPC 方法做特殊认证或校验。今天将会讲解、实现这块的功能点

## 课前知识

```
type PerRPCCredentials interface {
    GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error)
    RequireTransportSecurity() bool
}
```

在 gRPC 中默认定义了 PerRPCCredentials，它就是本章节的主角，是 gRPC 默认提供用于自定义认证的接口，它的作用是将所需的安全认证信息添加到每个 RPC 方法的上下文中。其包含 2 个方法：

- GetRequestMetadata：获取当前请求认证所需的元数据（metadata）
- RequireTransportSecurity：是否需要基于 TLS 认证进行安全传输

## 目录结构

新建 simple_token_server/server.go 和 simple_token_client/client.go，目录结构如下：

```
go-grpc-example
├── client
│   ├── simple_client
│   ├── simple_http_client
│   ├── simple_token_client
│   └── stream_client
├── conf
├── pkg
├── proto
├── server
│   ├── simple_http_server
│   ├── simple_server
│   ├── simple_token_server
│   └── stream_server
└── vendor
```

## gRPC

### Client

```
package main

import (
	"context"
	"log"

	"google.golang.org/grpc"

	"github.com/EDDYCJY/go-grpc-example/pkg/gtls"
	pb "github.com/EDDYCJY/go-grpc-example/proto"
)

const PORT = "9004"

type Auth struct {
	AppKey    string
	AppSecret string
}

func (a *Auth) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
	return map[string]string{"app_key": a.AppKey, "app_secret": a.AppSecret}, nil
}

func (a *Auth) RequireTransportSecurity() bool {
	return true
}

func main() {
	tlsClient := gtls.Client{
		ServerName: "go-grpc-example",
		CertFile:   "../../conf/server/server.pem",
	}
	c, err := tlsClient.GetTLSCredentials()
	if err != nil {
		log.Fatalf("tlsClient.GetTLSCredentials err: %v", err)
	}

	auth := Auth{
		AppKey:    "eddycjy",
		AppSecret: "20181005",
	}
	conn, err := grpc.Dial(":"+PORT, grpc.WithTransportCredentials(c), grpc.WithPerRPCCredentials(&auth))
	...
}
```

在 Client 端，重点实现 `type PerRPCCredentials interface` 所需的方法，关注两点即可：

- struct Auth：GetRequestMetadata、RequireTransportSecurity
- grpc.WithPerRPCCredentials

### Server

```
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"

	"github.com/EDDYCJY/go-grpc-example/pkg/gtls"
	pb "github.com/EDDYCJY/go-grpc-example/proto"
)

type SearchService struct {
	auth *Auth
}

func (s *SearchService) Search(ctx context.Context, r *pb.SearchRequest) (*pb.SearchResponse, error) {
	if err := s.auth.Check(ctx); err != nil {
		return nil, err
	}
	return &pb.SearchResponse{Response: r.GetRequest() + " Token Server"}, nil
}

const PORT = "9004"

func main() {
	...
}

type Auth struct {
	appKey    string
	appSecret string
}

func (a *Auth) Check(ctx context.Context) error {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return status.Errorf(codes.Unauthenticated, "自定义认证 Token 失败")
	}

	var (
		appKey    string
		appSecret string
	)
	if value, ok := md["app_key"]; ok {
		appKey = value[0]
	}
	if value, ok := md["app_secret"]; ok {
		appSecret = value[0]
	}

	if appKey != a.GetAppKey() || appSecret != a.GetAppSecret() {
		return status.Errorf(codes.Unauthenticated, "自定义认证 Token 无效")
	}

	return nil
}

func (a *Auth) GetAppKey() string {
	return "eddycjy"
}

func (a *Auth) GetAppSecret() string {
	return "20181005"
}
```

在 Server 端就更简单了，实际就是调用 `metadata.FromIncomingContext` 从上下文中获取 metadata，再在不同的 RPC 方法中进行认证检查

### 验证

重新启动 server.go 和 client.go，得到以下结果：

```
$ go run client.go
2018/10/05 20:59:58 resp: gRPC Token Server
```

修改 client.go 的值，制造两者不一致，得到无效结果：

```
$ go run client.go
2018/10/05 21:00:05 client.Search err: rpc error: code = Unauthenticated desc = invalid token
exit status 1
```

### 一个个加太麻烦

我相信你肯定会问一个个加，也太麻烦了吧？有这个想法的你，应当把 `type PerRPCCredentials interface` 做成一个拦截器（interceptor）

## 总结

本章节比较简单，主要是针对 RPC 方法的自定义认证进行了介绍，如果是想做全局的，建议是举一反三从拦截器下手哦 😄

## 参考
### 本系列示例代码
- [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)
