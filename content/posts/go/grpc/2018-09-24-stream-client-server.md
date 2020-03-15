---

title:      "「连载三」gRPC Streaming, Client and Server"
date:       2018-09-24 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - grpc
---

## 前言

本章节将介绍 gRPC 的流式，分为三种类型：

- Server-side streaming RPC：服务器端流式 RPC
- Client-side streaming RPC：客户端流式 RPC
- Bidirectional streaming RPC：双向流式 RPC

## 流

任何技术，因为有痛点，所以才有了存在的必要性。如果您想要了解 gRPC 的流式调用，请继续

### 图

![image](https://image.eddycjy.com/8812038d20ffece377c0e4901c9a9231.png)

gRPC Streaming 是基于 HTTP/2 的，后续章节再进行详细讲解

### 为什么不用 Simple RPC

流式为什么要存在呢，是 Simple RPC 有什么问题吗？通过模拟业务场景，可得知在使用 Simple RPC 时，有如下问题：

- 数据包过大造成的瞬时压力
- 接收数据包时，需要所有数据包都接受成功且正确后，才能够回调响应，进行业务处理（无法客户端边发送，服务端边处理）

### 为什么用 Streaming RPC

- 大规模数据包
- 实时场景

#### 模拟场景

每天早上 6 点，都有一批百万级别的数据集要同从 A 同步到 B，在同步的时候，会做一系列操作（归档、数据分析、画像、日志等）。这一次性涉及的数据量确实大

在同步完成后，也有人马上会去查阅数据，为了新的一天筹备。也符合实时性。

两者相较下，这个场景下更适合使用 Streaming RPC

## gRPC

在讲解具体的 gRPC 流式代码时，会**着重在第一节讲解**，因为三种模式其实是不同的组合。希望你能够注重理解，举一反三，其实都是一样的知识点 👍

### 目录结构

```
$ tree go-grpc-example
go-grpc-example
├── client
│   ├── simple_client
│   │   └── client.go
│   └── stream_client
│       └── client.go
├── proto
│   ├── search.proto
│   └── stream.proto
└── server
    ├── simple_server
    │   └── server.go
    └── stream_server
        └── server.go
```

增加 stream_server、stream_client 存放服务端和客户端文件，proto/stream.proto 用于编写 IDL

### IDL

在 proto 文件夹下的 stream.proto 文件中，写入如下内容：

```
syntax = "proto3";

package proto;

service StreamService {
    rpc List(StreamRequest) returns (stream StreamResponse) {};

    rpc Record(stream StreamRequest) returns (StreamResponse) {};

    rpc Route(stream StreamRequest) returns (stream StreamResponse) {};
}


message StreamPoint {
  string name = 1;
  int32 value = 2;
}

message StreamRequest {
  StreamPoint pt = 1;
}

message StreamResponse {
  StreamPoint pt = 1;
}
```

注意关键字 stream，声明其为一个流方法。这里共涉及三个方法，对应关系为

- List：服务器端流式 RPC
- Record：客户端流式 RPC
- Route：双向流式 RPC

### 基础模板 + 空定义

#### Server

```go
package main

import (
	"log"
	"net"

	"google.golang.org/grpc"

	pb "github.com/EDDYCJY/go-grpc-example/proto"

)

type StreamService struct{}

const (
	PORT = "9002"
)

func main() {
	server := grpc.NewServer()
	pb.RegisterStreamServiceServer(server, &StreamService{})

	lis, err := net.Listen("tcp", ":"+PORT)
	if err != nil {
		log.Fatalf("net.Listen err: %v", err)
	}

	server.Serve(lis)
}

func (s *StreamService) List(r *pb.StreamRequest, stream pb.StreamService_ListServer) error {
	return nil
}

func (s *StreamService) Record(stream pb.StreamService_RecordServer) error {
	return nil
}

func (s *StreamService) Route(stream pb.StreamService_RouteServer) error {
	return nil
}
```

写代码前，建议先将 gRPC Server 的基础模板和接口给空定义出来。若有不清楚可参见上一章节的知识点

#### Client

```go
package main

import (
    "log"

	"google.golang.org/grpc"

	pb "github.com/EDDYCJY/go-grpc-example/proto"
)

const (
	PORT = "9002"
)

func main() {
	conn, err := grpc.Dial(":"+PORT, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("grpc.Dial err: %v", err)
	}

	defer conn.Close()

	client := pb.NewStreamServiceClient(conn)

	err = printLists(client, &pb.StreamRequest{Pt: &pb.StreamPoint{Name: "gRPC Stream Client: List", Value: 2018}})
	if err != nil {
		log.Fatalf("printLists.err: %v", err)
	}

	err = printRecord(client, &pb.StreamRequest{Pt: &pb.StreamPoint{Name: "gRPC Stream Client: Record", Value: 2018}})
	if err != nil {
		log.Fatalf("printRecord.err: %v", err)
	}

	err = printRoute(client, &pb.StreamRequest{Pt: &pb.StreamPoint{Name: "gRPC Stream Client: Route", Value: 2018}})
	if err != nil {
		log.Fatalf("printRoute.err: %v", err)
	}
}

func printLists(client pb.StreamServiceClient, r *pb.StreamRequest) error {
	return nil
}

func printRecord(client pb.StreamServiceClient, r *pb.StreamRequest) error {
	return nil
}

func printRoute(client pb.StreamServiceClient, r *pb.StreamRequest) error {
	return nil
}
```

### 一、Server-side streaming RPC：服务器端流式 RPC

服务器端流式 RPC，显然是单向流，并代指 Server 为 Stream 而 Client 为普通 RPC 请求

简单来讲就是客户端发起一次普通的 RPC 请求，服务端通过流式响应多次发送数据集，客户端 Recv 接收数据集。大致如图：

![image](https://image.eddycjy.com/b25a47e2f2fb2a8c352a547f7612808b.png)

#### Server

```go
func (s *StreamService) List(r *pb.StreamRequest, stream pb.StreamService_ListServer) error {
	for n := 0; n <= 6; n++ {
		err := stream.Send(&pb.StreamResponse{
			Pt: &pb.StreamPoint{
				Name:  r.Pt.Name,
				Value: r.Pt.Value + int32(n),
			},
		})
		if err != nil {
			return err
		}
	}

	return nil
}
```

在 Server，主要留意 `stream.Send` 方法。它看上去能发送 N 次？有没有大小限制？

```go
type StreamService_ListServer interface {
	Send(*StreamResponse) error
	grpc.ServerStream
}

func (x *streamServiceListServer) Send(m *StreamResponse) error {
	return x.ServerStream.SendMsg(m)
}
```

通过阅读源码，可得知是 protoc 在生成时，根据定义生成了各式各样符合标准的接口方法。最终再统一调度内部的 `SendMsg` 方法，该方法涉及以下过程:

- 消息体（对象）序列化
- 压缩序列化后的消息体
- 对正在传输的消息体增加 5 个字节的 header
- 判断压缩+序列化后的消息体总字节长度是否大于预设的 maxSendMessageSize（预设值为 `math.MaxInt32`），若超出则提示错误
- 写入给流的数据集

#### Client

```go
func printLists(client pb.StreamServiceClient, r *pb.StreamRequest) error {
	stream, err := client.List(context.Background(), r)
	if err != nil {
		return err
	}

	for {
		resp, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			return err
		}

		log.Printf("resp: pj.name: %s, pt.value: %d", resp.Pt.Name, resp.Pt.Value)
	}

	return nil
}
```

在 Client，主要留意 `stream.Recv()` 方法。什么情况下 `io.EOF` ？什么情况下存在错误信息呢?

```go
type StreamService_ListClient interface {
	Recv() (*StreamResponse, error)
	grpc.ClientStream
}

func (x *streamServiceListClient) Recv() (*StreamResponse, error) {
	m := new(StreamResponse)
	if err := x.ClientStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
```

RecvMsg 会从流中读取完整的 gRPC 消息体，另外通过阅读源码可得知：

（1）RecvMsg 是阻塞等待的

（2）RecvMsg 当流成功/结束（调用了 Close）时，会返回 `io.EOF`

（3）RecvMsg 当流出现任何错误时，流会被中止，错误信息会包含 RPC 错误码。而在 RecvMsg 中可能出现如下错误：

- io.EOF
- io.ErrUnexpectedEOF
- transport.ConnectionError
- google.golang.org/grpc/codes

同时需要注意，默认的 MaxReceiveMessageSize 值为 1024 _ 1024 _ 4，建议不要超出

#### 验证

运行 stream_server/server.go：

```
$ go run server.go
```

运行 stream_client/client.go：

```
$ go run client.go
2018/09/24 16:18:25 resp: pj.name: gRPC Stream Client: List, pt.value: 2018
2018/09/24 16:18:25 resp: pj.name: gRPC Stream Client: List, pt.value: 2019
2018/09/24 16:18:25 resp: pj.name: gRPC Stream Client: List, pt.value: 2020
2018/09/24 16:18:25 resp: pj.name: gRPC Stream Client: List, pt.value: 2021
2018/09/24 16:18:25 resp: pj.name: gRPC Stream Client: List, pt.value: 2022
2018/09/24 16:18:25 resp: pj.name: gRPC Stream Client: List, pt.value: 2023
2018/09/24 16:18:25 resp: pj.name: gRPC Stream Client: List, pt.value: 2024
```

### 二、Client-side streaming RPC：客户端流式 RPC

客户端流式 RPC，单向流，客户端通过流式发起**多次** RPC 请求给服务端，服务端发起**一次**响应给客户端，大致如图：

![image](https://image.eddycjy.com/97473884d939ec91d6cdf53090bef92e.png)

#### Server

```go
func (s *StreamService) Record(stream pb.StreamService_RecordServer) error {
	for {
		r, err := stream.Recv()
		if err == io.EOF {
			return stream.SendAndClose(&pb.StreamResponse{Pt: &pb.StreamPoint{Name: "gRPC Stream Server: Record", Value: 1}})
		}
		if err != nil {
			return err
		}

		log.Printf("stream.Recv pt.name: %s, pt.value: %d", r.Pt.Name, r.Pt.Value)
	}

	return nil
}
```

多了一个从未见过的方法 `stream.SendAndClose`，它是做什么用的呢？

在这段程序中，我们对每一个 Recv 都进行了处理，当发现 `io.EOF` (流关闭) 后，需要将最终的响应结果发送给客户端，同时关闭正在另外一侧等待的 Recv

#### Client

```go
func printRecord(client pb.StreamServiceClient, r *pb.StreamRequest) error {
	stream, err := client.Record(context.Background())
	if err != nil {
		return err
	}

	for n := 0; n < 6; n++ {
		err := stream.Send(r)
		if err != nil {
			return err
		}
	}

	resp, err := stream.CloseAndRecv()
	if err != nil {
		return err
	}

	log.Printf("resp: pj.name: %s, pt.value: %d", resp.Pt.Name, resp.Pt.Value)

	return nil
}
```

`stream.CloseAndRecv` 和 `stream.SendAndClose` 是配套使用的流方法，相信聪明的你已经秒懂它的作用了

#### 验证

重启 stream_server/server.go，再次运行 stream_client/client.go：

##### stream_client：

```
$ go run client.go
2018/09/24 16:23:03 resp: pj.name: gRPC Stream Server: Record, pt.value: 1
```

##### stream_server：

```
$ go run server.go
2018/09/24 16:23:03 stream.Recv pt.name: gRPC Stream Client: Record, pt.value: 2018
2018/09/24 16:23:03 stream.Recv pt.name: gRPC Stream Client: Record, pt.value: 2018
2018/09/24 16:23:03 stream.Recv pt.name: gRPC Stream Client: Record, pt.value: 2018
2018/09/24 16:23:03 stream.Recv pt.name: gRPC Stream Client: Record, pt.value: 2018
2018/09/24 16:23:03 stream.Recv pt.name: gRPC Stream Client: Record, pt.value: 2018
2018/09/24 16:23:03 stream.Recv pt.name: gRPC Stream Client: Record, pt.value: 2018
```

### 三、Bidirectional streaming RPC：双向流式 RPC

双向流式 RPC，顾名思义是双向流。由客户端以流式的方式发起请求，服务端同样以流式的方式响应请求

首个请求一定是 Client 发起，但具体交互方式（谁先谁后、一次发多少、响应多少、什么时候关闭）根据程序编写的方式来确定（可以结合协程）

假设该双向流是**按顺序发送**的话，大致如图：

![image](https://image.eddycjy.com/ab80297cd6715048a235e0c9b0f36091.png)

还是要强调，双向流变化很大，因程序编写的不同而不同。**双向流图示无法适用不同的场景**

#### Server

```go
func (s *StreamService) Route(stream pb.StreamService_RouteServer) error {
	n := 0
	for {
		err := stream.Send(&pb.StreamResponse{
			Pt: &pb.StreamPoint{
				Name:  "gPRC Stream Client: Route",
				Value: int32(n),
			},
		})
		if err != nil {
			return err
		}

		r, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}

		n++

		log.Printf("stream.Recv pt.name: %s, pt.value: %d", r.Pt.Name, r.Pt.Value)
	}

	return nil
}
```

#### Client

```go
func printRoute(client pb.StreamServiceClient, r *pb.StreamRequest) error {
	stream, err := client.Route(context.Background())
	if err != nil {
		return err
	}

	for n := 0; n <= 6; n++ {
		err = stream.Send(r)
		if err != nil {
			return err
		}

		resp, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			return err
		}

		log.Printf("resp: pj.name: %s, pt.value: %d", resp.Pt.Name, resp.Pt.Value)
	}

	stream.CloseSend()

	return nil
}
```

#### 验证

重启 stream_server/server.go，再次运行 stream_client/client.go：

##### stream_server

```
$ go run server.go
2018/09/24 16:29:43 stream.Recv pt.name: gRPC Stream Client: Route, pt.value: 2018
2018/09/24 16:29:43 stream.Recv pt.name: gRPC Stream Client: Route, pt.value: 2018
2018/09/24 16:29:43 stream.Recv pt.name: gRPC Stream Client: Route, pt.value: 2018
2018/09/24 16:29:43 stream.Recv pt.name: gRPC Stream Client: Route, pt.value: 2018
2018/09/24 16:29:43 stream.Recv pt.name: gRPC Stream Client: Route, pt.value: 2018
2018/09/24 16:29:43 stream.Recv pt.name: gRPC Stream Client: Route, pt.value: 2018
```

##### stream_client

```
$ go run client.go
2018/09/24 16:29:43 resp: pj.name: gPRC Stream Client: Route, pt.value: 0
2018/09/24 16:29:43 resp: pj.name: gPRC Stream Client: Route, pt.value: 1
2018/09/24 16:29:43 resp: pj.name: gPRC Stream Client: Route, pt.value: 2
2018/09/24 16:29:43 resp: pj.name: gPRC Stream Client: Route, pt.value: 3
2018/09/24 16:29:43 resp: pj.name: gPRC Stream Client: Route, pt.value: 4
2018/09/24 16:29:43 resp: pj.name: gPRC Stream Client: Route, pt.value: 5
2018/09/24 16:29:43 resp: pj.name: gPRC Stream Client: Route, pt.value: 6
```

## 总结

在本文共介绍了三类流的交互方式，可以根据实际的业务场景去选择合适的方式。会事半功倍哦 🎑

## 参考

### 本系列示例代码

- [go-grpc-example](https://github.com/EDDYCJY/go-grpc-example)