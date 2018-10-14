# 带入gRPC：分布式链路追踪 gRPC + Opentracing + Zipkin

在实际应用中，你做了那么多 Server 端，写了 N 个 RPC 方法。想看看方法的指标，却无处下手？

本文将通过 gRPC + Opentracing + Zipkin 搭建一个**分布式链路追踪系统**来实现查看整个系统的链路、性能等指标 🤓

## Opentracing

### 是什么

OpenTracing 通过提供平台无关、厂商无关的API，使得开发人员能够方便的添加（或更换）追踪系统的实现

不过 OpenTracing 并不是标准。因为 CNCF 不是官方标准机构，但是它的目标是致力为分布式追踪创建更标准的 API 和工具

### 名词解释

#### Trace

一个 trace 代表了一个事务或者流程在（分布式）系统中的执行过程

#### Span

一个 span 代表在分布式系统中完成的单个工作单元。也包含其他 span 的 “引用”，这允许将多个 spans 组合成一个完整的 Trace

每个 span 根据 OpenTracing 规范封装以下内容：

- 操作名称
- 开始时间和结束时间
- key:value span Tags
- key:value span Logs
- SpanContext

#### Tags

Span tags（跨度标签）可以理解为用户自定义的 Span 注释。便于查询、过滤和理解跟踪数据

#### Logs

Span logs（跨度日志）可以记录 Span 内特定时间或事件的日志信息。主要用于捕获特定 Span 的日志信息以及应用程序本身的其他调试或信息输出

#### SpanContext

SpanContext 代表跨越进程边界，传递到子级 Span 的状态。常在追踪示意图中创建上下文时使用

#### Baggage Items

Baggage Items 可以理解为 trace 全局运行中额外传输的数据集合

### 一个案例

![image](https://wu-sheng.gitbooks.io/opentracing-io/content/images/OTOV_3.png)

图中可以看到以下内容：

- 执行时间的上下文
- 服务间的层次关系
- 服务间串行或并行调用链

结合以上信息，在实际场景中我们可以通过整个系统的调用链的上下文、性能等指标信息，一下子就能够发现系统的痛点在哪儿 

## Zipkin

![image](https://i.imgur.com/rZU6zoj.png)

### 是什么

Zipkin 是分布式追踪系统。它的作用是收集解决微服务架构中的延迟问题所需的时序数据。它管理这些数据的收集和查找

Zipkin 的设计基于 [Google Dapper](http://research.google.com/pubs/pub36356.html) 论文。

### 运行

```
docker run -d -p 9411:9411 openzipkin/zipkin
```

其他方法安装参见：https://github.com/openzipkin/zipkin

### 验证

访问 http://127.0.0.1:9411/zipkin/ 检查 Zipkin 是否运行正常

![image](https://i.imgur.com/iWhfEef.jpg)

## gRPC + Opentracing + Zipkin

在前面的小节中，我们做了以下准备工作：

- 了解 Opentracing 是什么
- 搭建 Zipkin 提供分布式追踪系统的功能

接下来实现 gRPC 通过 Opentracing 标准 API 对接 Zipkin，再通过 Zipkin 去查看数据

### 目录结构

新建 simple_zipkin_client、simple_zipkin_server 目录，目录结构如下：

```
go-grpc-example
├── LICENSE
├── README.md
├── client
│   ├── ...
│   ├── simple_zipkin_client
├── conf
├── pkg
├── proto
├── server
│   ├── ...
│   ├── simple_zipkin_server
└── vendor
```

### 安装

```
$ go get -u github.com/openzipkin/zipkin-go-opentracing
$ go get -u github.com/grpc-ecosystem/grpc-opentracing/go/otgrpc
```

### gRPC

#### Server

```
package main

import (
	"context"
	"log"
	"net"

	"github.com/grpc-ecosystem/go-grpc-middleware"
	"github.com/grpc-ecosystem/grpc-opentracing/go/otgrpc"
	zipkin "github.com/openzipkin/zipkin-go-opentracing"
	"google.golang.org/grpc"

	"github.com/EDDYCJY/go-grpc-example/pkg/gtls"
	pb "github.com/EDDYCJY/go-grpc-example/proto"
)

type SearchService struct{}

func (s *SearchService) Search(ctx context.Context, r *pb.SearchRequest) (*pb.SearchResponse, error) {
	return &pb.SearchResponse{Response: r.GetRequest() + " Server"}, nil
}

const (
	PORT = "9005"

	SERVICE_NAME              = "simple_zipkin_server"
	ZIPKIN_HTTP_ENDPOINT      = "http://127.0.0.1:9411/api/v1/spans"
	ZIPKIN_RECORDER_HOST_PORT = "127.0.0.1:9000"
)

func main() {
	collector, err := zipkin.NewHTTPCollector(ZIPKIN_HTTP_ENDPOINT)
	if err != nil {
		log.Fatalf("zipkin.NewHTTPCollector err: %v", err)
	}

	recorder := zipkin.NewRecorder(collector, true, ZIPKIN_RECORDER_HOST_PORT, SERVICE_NAME)

	tracer, err := zipkin.NewTracer(
		recorder, zipkin.ClientServerSameSpan(false),
	)
	if err != nil {
		log.Fatalf("zipkin.NewTracer err: %v", err)
	}

	tlsServer := gtls.Server{
		CaFile:   "../../conf/ca.pem",
		CertFile: "../../conf/server/server.pem",
		KeyFile:  "../../conf/server/server.key",
	}
	c, err := tlsServer.GetCredentialsByCA()
	if err != nil {
		log.Fatalf("GetTLSCredentialsByCA err: %v", err)
	}

	opts := []grpc.ServerOption{
		grpc.Creds(c),
		grpc_middleware.WithUnaryServerChain(
			otgrpc.OpenTracingServerInterceptor(tracer, otgrpc.LogPayloads()),
		),
	}
    ...
}
```

- zipkin.NewHTTPCollector：创建一个 Zipkin HTTP 后端收集器  
- zipkin.NewRecorder：创建一个基于 Zipkin 收集器的记录器
- zipkin.NewTracer：创建一个 OpenTracing 跟踪器（兼容 Zipkin Tracer）
- otgrpc.OpenTracingClientInterceptor：返回 grpc.UnaryServerInterceptor，不同点在于该拦截器会在 gRPC Metadata 中查找 OpenTracing SpanContext。如果找到则为该服务的 Span Context 的子节点 
- otgrpc.LogPayloads：设置并返回 Option。作用是让 OpenTracing 在双向方向上记录应用程序的有效载荷（payload）

总的来讲，就是初始化 Zipkin，其又包含收集器、记录器、跟踪器。再利用拦截器在 Server 端实现 SpanContext、Payload 的双向读取和管理

#### Client

```
func main() {
	// the same as zipkin server
	// ...
	conn, err := grpc.Dial(":"+PORT, grpc.WithTransportCredentials(c),
		grpc.WithUnaryInterceptor(
			otgrpc.OpenTracingClientInterceptor(tracer, otgrpc.LogPayloads()),
		))
	...
}
```

- otgrpc.OpenTracingClientInterceptor：返回 grpc.UnaryClientInterceptor。该拦截器的核心功能在于：

（1）OpenTracing SpanContext 注入 gRPC Metadata 

（2）查看 context.Context 中的上下文关系，若存在父级 Span 则创建一个 ChildOf 引用，得到一个子 Span

其他方面，与 Server 端是一致的，先初始化 Zipkin，再增加 Client 端特需的拦截器。就可以完成基础工作啦

### 验证

启动 Server.go，执行 Client.go。查看 http://127.0.0.1:9411/zipkin/ 的示意图：

![image](https://i.imgur.com/z2IRxnj.jpg)

![image](https://i.imgur.com/0rqEzvl.jpg)

## 复杂点

![image](https://i.imgur.com/0Nuq66Z.jpg)

![image](https://i.imgur.com/eRr62ny.jpg)

## 总结

在多服务下的架构下，串行、并行、服务套服务是一个非常常见的情况，用常规的方案往往很难发现问题在哪里（成本太大）。而这种情况就是**分布式追踪系统**大展拳脚的机会了

希望你通过本章节的介绍和学习，能够了解其概念和搭建且应用一个追踪系统 😄

## 参考
### 本系列示例代码
- [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)

### 资料
- [opentracing](https://opentracing.io/)
- [zipkin](https://zipkin.io)

