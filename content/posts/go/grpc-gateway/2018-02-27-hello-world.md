---

title:      "「连载二」Hello World"
date:       2018-02-27 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - grpc-gateway
---

这节将开始编写一个复杂的Hello World，涉及到许多的知识，建议大家认真思考其中的概念

## 需求
由于本实践偏向`Grpc`+`Grpc Gateway`的方面，我们的需求是**同一个服务端支持`Rpc`和`Restful Api`**，那么就意味着`http2`、`TLS`等等的应用，功能方面就是一个服务端能够接受来自`grpc`和`Restful Api`的请求并响应

## 一、初始化目录

我们先在$GOPATH中新建`grpc-hello-world`文件夹，我们项目的初始目录目录如下：
```
grpc-hello-world/
├── certs
├── client
├── cmd
├── pkg
├── proto
│   ├── google
│   │   └── api
└── server
```
- `certs`：证书凭证
- `client`：客户端
- `cmd`：命令行
- `pkg`：第三方公共模块
- `proto`：`protobuf`的一些相关文件（含`.proto`、`pb.go`、`.pb.gw.go`)，`google/api`中用于存放`annotations.proto`、`http.proto`
- `server`：服务端

## 二、制作证书

在服务端支持`Rpc`和`Restful Api`，需要用到`TLS`，因此我们要先制作证书

进入`certs`目录，生成`TLS`所需的公钥密钥文件

### 私钥
```
openssl genrsa -out server.key 2048

openssl ecparam -genkey -name secp384r1 -out server.key
```
- `openssl genrsa`：生成`RSA`私钥，命令的最后一个参数，将指定生成密钥的位数，如果没有指定，默认512
- `openssl ecparam`：生成`ECC`私钥，命令为椭圆曲线密钥参数生成及操作，本文中`ECC`曲线选择的是`secp384r1`

### 自签名公钥
```
openssl req -new -x509 -sha256 -key server.key -out server.pem -days 3650
```
- `openssl req`：生成自签名证书，`-new`指生成证书请求、`-sha256`指使用`sha256`加密、`-key`指定私钥文件、`-x509`指输出证书、`-days 3650`为有效期，此后则输入证书拥有者信息

### 填写信息
```
Country Name (2 letter code) [XX]:
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:grpc server name
Email Address []:
```
## 三、`proto`
### 编写

1、 `google.api`

我们看到`proto`目录中有`google/api`目录，它用到了`google`官方提供的两个`api`描述文件，主要是针对`grpc-gateway`的`http`转换提供支持，定义了`Protocol Buffer`所扩展的`HTTP Option`

`annotations.proto`文件：
```
// Copyright (c) 2015, Google Inc.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

syntax = "proto3";

package google.api;

import "google/api/http.proto";
import "google/protobuf/descriptor.proto";

option java_multiple_files = true;
option java_outer_classname = "AnnotationsProto";
option java_package = "com.google.api";

extend google.protobuf.MethodOptions {
  // See `HttpRule`.
  HttpRule http = 72295728;
}

```

`http.proto`文件：
```
// Copyright 2016 Google Inc.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

syntax = "proto3";

package google.api;

option cc_enable_arenas = true;
option java_multiple_files = true;
option java_outer_classname = "HttpProto";
option java_package = "com.google.api";


// Defines the HTTP configuration for a service. It contains a list of
// [HttpRule][google.api.HttpRule], each specifying the mapping of an RPC method
// to one or more HTTP REST API methods.
message Http {
  // A list of HTTP rules for configuring the HTTP REST API methods.
  repeated HttpRule rules = 1;
}

// Use CustomHttpPattern to specify any HTTP method that is not included in the
// `pattern` field, such as HEAD, or "*" to leave the HTTP method unspecified for
// a given URL path rule. The wild-card rule is useful for services that provide
// content to Web (HTML) clients.
message HttpRule {
  // Selects methods to which this rule applies.
  //
  // Refer to [selector][google.api.DocumentationRule.selector] for syntax details.
  string selector = 1;

  // Determines the URL pattern is matched by this rules. This pattern can be
  // used with any of the {get|put|post|delete|patch} methods. A custom method
  // can be defined using the 'custom' field.
  oneof pattern {
    // Used for listing and getting information about resources.
    string get = 2;

    // Used for updating a resource.
    string put = 3;

    // Used for creating a resource.
    string post = 4;

    // Used for deleting a resource.
    string delete = 5;

    // Used for updating a resource.
    string patch = 6;

    // Custom pattern is used for defining custom verbs.
    CustomHttpPattern custom = 8;
  }

  // The name of the request field whose value is mapped to the HTTP body, or
  // `*` for mapping all fields not captured by the path pattern to the HTTP
  // body. NOTE: the referred field must not be a repeated field.
  string body = 7;

  // Additional HTTP bindings for the selector. Nested bindings must
  // not contain an `additional_bindings` field themselves (that is,
  // the nesting may only be one level deep).
  repeated HttpRule additional_bindings = 11;
}

// A custom pattern is used for defining custom HTTP verb.
message CustomHttpPattern {
  // The name of this custom HTTP verb.
  string kind = 1;

  // The path matched by this custom verb.
  string path = 2;
}

```

2. `hello.proto`

这一小节将编写`Demo`的`.proto`文件，我们在`proto`目录下新建`hello.proto`文件，写入文件内容：
```
syntax = "proto3";

package proto;

import "google/api/annotations.proto";

service HelloWorld {
    rpc SayHelloWorld(HelloWorldRequest) returns (HelloWorldResponse) {
        option (google.api.http) = {
            post: "/hello_world"
            body: "*"
        };
    }
}

message HelloWorldRequest {
    string referer = 1;
}

message HelloWorldResponse {
    string message = 1;
}
```

在`hello.proto`文件中，引用了`google/api/annotations.proto`，达到支持`HTTP Option`的效果

- 定义了一个`service`RPC服务`HelloWorld`，在其内部定义了一个`HTTP Option`的`POST`方法，`HTTP`响应路径为`/hello_world`
- 定义`message`类型`HelloWorldRequest`、`HelloWorldResponse`，用于响应请求和返回结果

### 编译

在编写完`.proto`文件后，我们需要对其进行编译，就能够在`server`中使用

进入`proto`目录，执行以下命令

```
# 编译google.api
protoc -I . --go_out=plugins=grpc,Mgoogle/protobuf/descriptor.proto=github.com/golang/protobuf/protoc-gen-go/descriptor:. google/api/*.proto

#编译hello_http.proto为hello_http.pb.proto
protoc -I . --go_out=plugins=grpc,Mgoogle/api/annotations.proto=grpc-hello-world/proto/google/api:. ./hello.proto

#编译hello_http.proto为hello_http.pb.gw.proto
protoc --grpc-gateway_out=logtostderr=true:. ./hello.proto
```

执行完毕后将生成`hello.pb.go`和`hello.gw.pb.go`，分别针对`grpc`和`grpc-gateway`的功能支持


## 四、命令行模块 `cmd`
### 介绍
这一小节我们编写命令行模块，为什么要独立出来呢，是为了将`cmd`和`server`两者解耦，避免混淆在一起。

我们采用 [Cobra](https://github.com/spf13/cobra) 来完成这项功能，`Cobra`既是创建强大的现代CLI应用程序的库，也是生成应用程序和命令文件的程序。提供了以下功能：

- 简易的子命令行模式
- 完全兼容posix的命令行模式(包括短和长版本)
- 嵌套的子命令
- 全局、本地和级联`flags`
- 使用`Cobra`很容易的生成应用程序和命令，使用`cobra create appname`和`cobra add cmdname`
- 智能提示
- 自动生成commands和flags的帮助信息
- 自动生成详细的help信息`-h`，`--help`等等
- 自动生成的bash自动完成功能
- 为应用程序自动生成手册
- 命令别名
- 定义您自己的帮助、用法等的灵活性。
- 可选与[viper](https://github.com/spf13/viper)紧密集成的apps

### 编写`server`

在编写`cmd`时需要先用`server`进行测试关联，因此这一步我们先写`server.go`用于测试

在`server`模块下 新建`server.go`文件，写入测试内容：
```
package server

import (
    "log"
)

var (
    ServerPort string
    CertName string
    CertPemPath string
    CertKeyPath string
)

func Serve() (err error){
    log.Println(ServerPort)
    
    log.Println(CertName)
    
    log.Println(CertPemPath)
    
    log.Println(CertKeyPath)
    
    return nil
}

```
### 编写`cmd`

在`cmd`模块下 新建`root.go`文件，写入内容：
```
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "grpc",
    Short: "Run the gRPC hello-world server",
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Println(err)
        os.Exit(-1)
    }
}
```
新建`server.go`文件，写入内容：
```
package cmd

import (
	"log"

	"github.com/spf13/cobra"
	
	"grpc-hello-world/server"
)

var serverCmd = &cobra.Command{
	Use:   "server",
	Short: "Run the gRPC hello-world server",
	Run: func(cmd *cobra.Command, args []string) {
		defer func() {
			if err := recover(); err != nil {
				log.Println("Recover error : %v", err)
			}
		}()
		
		server.Serve()
	},
}

func init() {
	serverCmd.Flags().StringVarP(&server.ServerPort, "port", "p", "50052", "server port")
	serverCmd.Flags().StringVarP(&server.CertPemPath, "cert-pem", "", "./certs/server.pem", "cert pem path")
	serverCmd.Flags().StringVarP(&server.CertKeyPath, "cert-key", "", "./certs/server.key", "cert key path")
	serverCmd.Flags().StringVarP(&server.CertName, "cert-name", "", "grpc server name", "server's hostname")
	rootCmd.AddCommand(serverCmd)
}
```

我们在`grpc-hello-world/`目录下，新建文件`main.go`，写入内容：
```
package main

import (
	"grpc-hello-world/cmd"
)

func main() {
	cmd.Execute()
}
```

### 讲解

要使用`Cobra`，按照`Cobra`标准要创建`main.go`和一个`rootCmd`文件，另外我们有子命令`server`

1、`rootCmd`：
`rootCmd`表示在没有任何子命令的情况下的基本命令

2、`&cobra.Command`：
- `Use`：`Command`的用法，`Use`是一个行用法消息
- `Short`：`Short`是`help`命令输出中显示的简短描述
- `Run`：运行:典型的实际工作功能。大多数命令只会实现这一点；另外还有`PreRun`、`PreRunE`、`PostRun`、`PostRunE`等等不同时期的运行命令，但比较少用，具体使用时再查看亦可

3、`rootCmd.AddCommand`：`AddCommand`向这父命令（`rootCmd`）添加一个或多个命令

4、`serverCmd.Flags().StringVarP()`：

一般来说，我们需要在`init()`函数中定义`flags`和处理配置，以`serverCmd.Flags().StringVarP(&server.ServerPort, "port", "p", "50052", "server port")`为例，我们定义了一个`flag`，值存储在`&server.ServerPort`中，长命令为`--port`，短命令为`-p`，，默认值为`50052`，命令的描述为`server port`。这一种调用方式成为`Local Flags`

我们延伸一下，如果觉得每一个子命令都要设一遍觉得很麻烦，我们可以采用`Persistent Flags`：

`rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")`

作用：

`flag`是可以持久的，这意味着该`flag`将被分配给它所分配的命令以及该命令下的每个命令。对于全局标记，将标记作为根上的持久标志。

另外还有`Local Flag on Parent Commands`、`Bind Flags with Config`、`Required flags`等等，使用到再 [传送](https://github.com/spf13/cobra#local-flag-on-parent-commands) 了解即可


### 测试
回到`grpc-hello-world/`目录下执行`go run main.go server`，查看输出是否为（此时应为默认值）：
```
2018/02/25 23:23:21 50052
2018/02/25 23:23:21 dev
2018/02/25 23:23:21 ./certs/server.pem
2018/02/25 23:23:21 ./certs/server.key
```

执行`go run main.go server --port=8000 --cert-pem=test-pem --cert-key=test-key --cert-name=test-name`，检验命令行参数是否正确：
```
2018/02/25 23:24:56 8000
2018/02/25 23:24:56 test-name
2018/02/25 23:24:56 test-pem
2018/02/25 23:24:56 test-key
```

若都无误，那么恭喜你`cmd`模块的编写正确了，下一部分开始我们的重点章节！

## 五、服务端模块 `server`


### 编写`hello.go`
在`server`目录下新建文件`hello.go`，写入文件内容：
```
package server

import (
	"golang.org/x/net/context"

	pb "grpc-hello-world/proto"
)

type helloService struct{}

func NewHelloService() *helloService {
	return &helloService{}
}

func (h helloService) SayHelloWorld(ctx context.Context, r *pb.HelloWorldRequest) (*pb.HelloWorldResponse, error) {
	return &pb.HelloWorldResponse{
		Message : "test",
	}, nil
}
```

我们创建了`helloService`及其方法`SayHelloWorld`，对应`.proto`的`rpc SayHelloWorld`，这个方法需要有2个参数：`ctx context.Context`用于接受上下文参数、`r *pb.HelloWorldRequest`用于接受`protobuf`的`Request`参数（对应`.proto`的`message HelloWorldRequest`）


### *编写`server.go`
这一小章节，我们编写最为重要的服务端程序部分，涉及到大量的`grpc`、`grpc-gateway`及一些网络知识的应用

1、在`pkg`下新建`util`目录，新建`grpc.go`文件，写入内容：
```
package util

import (
	"net/http"
	"strings"

	"google.golang.org/grpc"
)

func GrpcHandlerFunc(grpcServer *grpc.Server, otherHandler http.Handler) http.Handler {
    if otherHandler == nil {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            grpcServer.ServeHTTP(w, r)
        })
    }
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.ProtoMajor == 2 && strings.Contains(r.Header.Get("Content-Type"), "application/grpc") {
            grpcServer.ServeHTTP(w, r)
        } else {
            otherHandler.ServeHTTP(w, r)
        }
    })
}
```

`GrpcHandlerFunc`函数是用于判断请求是来源于`Rpc`客户端还是`Restful Api`的请求，根据不同的请求注册不同的`ServeHTTP`服务；`r.ProtoMajor == 2`也代表着请求必须基于`HTTP/2`

2、在`pkg`下的`util`目录下，新建`tls.go`文件，写入内容：
```
package util

import (
	"crypto/tls"
    "io/ioutil"
    "log"

    "golang.org/x/net/http2"
)

func GetTLSConfig(certPemPath, certKeyPath string) *tls.Config {
    var certKeyPair *tls.Certificate
    cert, _ := ioutil.ReadFile(certPemPath)
    key, _ := ioutil.ReadFile(certKeyPath)
    
    pair, err := tls.X509KeyPair(cert, key)
    if err != nil {
        log.Println("TLS KeyPair err: %v\n", err)
    }
    
    certKeyPair = &pair

    return &tls.Config{
        Certificates: []tls.Certificate{*certKeyPair},
        NextProtos:   []string{http2.NextProtoTLS},
    }
}
```

`GetTLSConfig`函数是用于获取`TLS`配置，在内部，我们读取了`server.key`和`server.pem`这类证书凭证文件
- `tls.X509KeyPair`：从一对`PEM`编码的数据中解析公钥/私钥对。成功则返回公钥/私钥对
- `http2.NextProtoTLS`：`NextProtoTLS`是谈判期间的`NPN/ALPN`协议，用于**HTTP/2的TLS设置**
- `tls.Certificate`：返回一个或多个证书，实质我们解析`PEM`调用的`X509KeyPair`的函数声明就是`func X509KeyPair(certPEMBlock, keyPEMBlock []byte) (Certificate, error)`，返回值就是`Certificate`

总的来说该函数是用于处理从证书凭证文件（PEM），最终获取`tls.Config`作为`HTTP2`的使用参数

3、修改`server`目录下的`server.go`文件，该文件是我们服务里的核心文件，写入内容：
```
package server

import (
    "crypto/tls"
    "net"
    "net/http"
    "log"

    "golang.org/x/net/context"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    "github.com/grpc-ecosystem/grpc-gateway/runtime"
    
    pb "grpc-hello-world/proto"
    "grpc-hello-world/pkg/util"
)

var (
    ServerPort string
    CertName string
    CertPemPath string
    CertKeyPath string
    EndPoint string
)

func Serve() (err error){
    EndPoint = ":" + ServerPort
    conn, err := net.Listen("tcp", EndPoint)
    if err != nil {
        log.Printf("TCP Listen err:%v\n", err)
    }

    tlsConfig := util.GetTLSConfig(CertPemPath, CertKeyPath)
    srv := createInternalServer(conn, tlsConfig)

    log.Printf("gRPC and https listen on: %s\n", ServerPort)

    if err = srv.Serve(tls.NewListener(conn, tlsConfig)); err != nil {
        log.Printf("ListenAndServe: %v\n", err)
    }

    return err
}

func createInternalServer(conn net.Listener, tlsConfig *tls.Config) (*http.Server) {
    var opts []grpc.ServerOption

    // grpc server
    creds, err := credentials.NewServerTLSFromFile(CertPemPath, CertKeyPath)
    if err != nil {
        log.Printf("Failed to create server TLS credentials %v", err)
    }

    opts = append(opts, grpc.Creds(creds))
    grpcServer := grpc.NewServer(opts...)

    // register grpc pb
    pb.RegisterHelloWorldServer(grpcServer, NewHelloService())

    // gw server
    ctx := context.Background()
    dcreds, err := credentials.NewClientTLSFromFile(CertPemPath, CertName)
    if err != nil {
        log.Printf("Failed to create client TLS credentials %v", err)
    }
    dopts := []grpc.DialOption{grpc.WithTransportCredentials(dcreds)}
    gwmux := runtime.NewServeMux()

    // register grpc-gateway pb
    if err := pb.RegisterHelloWorldHandlerFromEndpoint(ctx, gwmux, EndPoint, dopts); err != nil {
        log.Printf("Failed to register gw server: %v\n", err)
    }

    // http服务
    mux := http.NewServeMux()
    mux.Handle("/", gwmux)

    return &http.Server{
        Addr:      EndPoint,
        Handler:   util.GrpcHandlerFunc(grpcServer, mux),
        TLSConfig: tlsConfig,
    }
}
```

#### `server`流程剖析

我们将这一大块代码，分成以下几个部分来理解

##### 一、启动监听

`net.Listen("tcp", EndPoint)`用于监听本地的网络地址通知，它的函数原型`func Listen(network, address string) (Listener, error)`

参数：`network`必须传入`tcp`、`tcp4`、`tcp6`、`unix`、`unixpacket`，若`address`为空或为0则会自动选择一个端口号
返回值：通过查看源码我们可以得知其返回值为`Listener`，结构体原型：
```
type Listener interface {
    Accept() (Conn, error)
    Close() error
    Addr() Addr
}
```

通过分析得知，**最后`net.Listen`会返回一个监听器的结构体，返回给接下来的动作，让其执行下一步的操作**，它可以执行三类操作
- `Accept`：接受等待并将下一个连接返回给`Listener`
- `Close`：关闭`Listener`
- `Addr`：返回`Listener`的网络地址

##### 二、获取`TLS`

通过`util.GetTLSConfig`解析得到`tls.Config`，传达给`http.Server`服务的`TLSConfig`配置项使用

##### 三、创建内部服务

`createInternalServer`函数，是整个服务端的核心流转部分

程序采用的是`HTT2`、`HTTPS`也就是需要支持`TLS`，因此在启动`grpc.NewServer`前，我们要将认证的中间件注册进去


而前面所获取的`tlsConfig`仅能给`HTTP`使用，因此**第一步**我们要创建`grpc`的`TLS`认证凭证

**1、创建`grpc`的`TLS`认证凭证**

新增引用`google.golang.org/grpc/credentials`的第三方包，它实现了`grpc`库支持的各种凭证，该凭证封装了客户机需要的所有状态，以便与服务器进行身份验证并进行各种断言，例如关于客户机的身份，角色或是否授权进行特定的呼叫

我们调用`NewServerTLSFromFile`来达到我们的目的，它能够从输入证书文件和服务器的密钥文件**构造TLS证书凭证**
```
func NewServerTLSFromFile(certFile, keyFile string) (TransportCredentials, error) {
    //LoadX509KeyPair读取并解析来自一对文件的公钥/私钥对
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, err
    }
    //NewTLS使用tls.Config来构建基于TLS的TransportCredentials
    return NewTLS(&tls.Config{Certificates: []tls.Certificate{cert}}), nil
}
```

**2、设置`grpc ServerOption`**

以`grpc.Creds(creds)`为例，其原型为`func Creds(c credentials.TransportCredentials) ServerOption`，该函数返回`ServerOption`，它为服务器连接设置凭据


**3、创建`grpc`服务端**

函数原型：
```
func NewServer(opt ...ServerOption) *Server
```
我们在此处创建了一个没有注册服务的`grpc`服务端，还没有开始接受请求
```
grpcServer := grpc.NewServer(opts...)
```
**4、注册`grpc`服务**
```
pb.RegisterHelloWorldServer(grpcServer, NewHelloService())
```

**5、创建`grpc-gateway`关联组件**
```
ctx := context.Background()
dcreds, err := credentials.NewClientTLSFromFile(CertPemPath, CertName)
if err != nil {
    log.Println("Failed to create client TLS credentials %v", err)
}
dopts := []grpc.DialOption{grpc.WithTransportCredentials(dcreds)}
```
- `context.Background`：返回一个非空的空上下文。它没有被注销，没有值，没有过期时间。它通常由主函数、初始化和测试使用，并作为传入请求的**顶级上下文**
- `credentials.NewClientTLSFromFile`：从客户机的输入证书文件构造TLS凭证
- `grpc.WithTransportCredentials`：配置一个连接级别的安全凭据(例：`TLS`、`SSL`)，返回值为`type DialOption`
- `grpc.DialOption`：`DialOption`选项配置我们如何设置连接（其内部具体由多个的`DialOption`组成，决定其设置连接的内容）

**6、创建`HTTP NewServeMux`及注册`grpc-gateway`逻辑**
```
gwmux := runtime.NewServeMux()

// register grpc-gateway pb
if err := pb.RegisterHelloWorldHandlerFromEndpoint(ctx, gwmux, EndPoint, dopts); err != nil {
    log.Println("Failed to register gw server: %v\n", err)
}

// http服务
mux := http.NewServeMux()
mux.Handle("/", gwmux)
```

- `runtime.NewServeMux`：返回一个新的`ServeMux`，它的内部映射是空的；`ServeMux`是`grpc-gateway`的一个请求多路复用器。它将`http`请求与模式匹配，并调用相应的处理程序
- `RegisterHelloWorldHandlerFromEndpoint`：如函数名，注册`HelloWorld`服务的`HTTP Handle`到`grpc`端点
- `http.NewServeMux`：`分配并返回一个新的ServeMux`
- `mux.Handle`：为给定模式注册处理程序

（带着疑问去看程序）为什么`gwmux`可以放入`mux.Handle`中？

首先我们看看它们的原型是怎么样的

（1）`http.NewServeMux()`
```
func NewServeMux() *ServeMux {
        return new(ServeMux) 
}
```
```
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```
（2）`runtime.NewServeMux`？
```
func NewServeMux(opts ...ServeMuxOption) *ServeMux {
    serveMux := &ServeMux{
        handlers:               make(map[string][]handler),
        forwardResponseOptions: make([]func(context.Context, http.ResponseWriter, proto.Message) error, 0),
        marshalers:             makeMarshalerMIMERegistry(),
    }
    ...
    return serveMux
}
```
（3）`http.NewServeMux()`的`Handle`方法
```
func (mux *ServeMux) Handle(pattern string, handler Handler)
```

通过分析可得知，两者`NewServeMux`都是最终返回`serveMux`，`Handler`中导出的方法仅有`ServeHTTP`，功能是用于响应HTTP请求

我们回到`Handle interface`中，可以得出结论就是任何结构体，只要实现了`ServeHTTP`方法，这个结构就可以称为`Handle`，`ServeMux`会使用该`Handler`调用`ServeHTTP`方法处理请求，这也就是**自定义`Handler`**

而我们这里正是将`grpc-gateway`中注册好的`HTTP Handler`无缝的植入到`net/http`的`Handle`方法中


**补充：在`go`中任何结构体只要实现了与接口相同的方法，就等同于实现了接口**

**7、注册具体服务**
```
if err := pb.RegisterHelloWorldHandlerFromEndpoint(ctx, gwmux, EndPoint, dopts); err != nil {
    log.Println("Failed to register gw server: %v\n", err)
}
```
在这段代码中，我们利用了前几小节的

- 上下文
- `gateway-grpc`的请求多路复用器
- 服务网络地址
- 配置好的安全凭据

注册了`HelloWorld`这一个服务

##### 四、创建`tls.NewListener`
```
func NewListener(inner net.Listener, config *Config) net.Listener {
    l := new(listener)
    l.Listener = inner
    l.config = config
    return l
}
```
`NewListener`将会创建一个`Listener`，它接受两个参数，第一个是来自内部`Listener`的监听器，第二个参数是`tls.Config`（必须包含至少一个证书）

##### 五、服务开始接受请求
在最后我们调用`srv.Serve(tls.NewListener(conn, tlsConfig))`，可以得知它是`http.Server`的方法，并且需要一个`Listener`作为参数，那么`Serve`内部做了些什么事呢？
```
func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()
    ...

    baseCtx := context.Background() // base is always background, per Issue 16220
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        rw, e := l.Accept()
        ...
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)
    }
}
```

粗略的看，它创建了一个`context.Background()`上下文对象，并调用`Listener`的`Accept`方法开始接受外部请求，在获取到连接数据后使用`newConn`创建连接对象，在最后使用`goroutine`的方式处理连接请求，达到其目的

**补充：对于`HTTP/2`支持，在调用`Serve`之前，应将`srv.TLSConfig`初始化为提供的`Listener`的TLS配置。如果`srv.TLSConfig`非零，并且在`Config.NextProtos`中不包含字符串`h2`，则不启用`HTTP/2`支持**

## 六、验证功能

### 编写测试客户端
在`grpc-hello-world/`下新建目录`client`，新建`client.go`文件，新增内容：
```
package main

import (
	"log"

	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"

	pb "grpc-hello-world/proto"
)

func main() {
	creds, err := credentials.NewClientTLSFromFile("../certs/server.pem", "dev")
	if err != nil {
		log.Println("Failed to create TLS credentials %v", err)
	}
	conn, err := grpc.Dial(":50052", grpc.WithTransportCredentials(creds))
	defer conn.Close()

	if err != nil {
		log.Println(err)
	}

	c := pb.NewHelloWorldClient(conn)
	context := context.Background()
	body := &pb.HelloWorldRequest{
		Referer : "Grpc",
	}

	r, err := c.SayHelloWorld(context, body)
	if err != nil {
		log.Println(err)
	}

	log.Println(r.Message)
}
```
由于客户端只是展示测试用，就简单的来了，原本它理应归类到`cobra`的管控下，配置管理等等都应可控化

在看这篇文章的你，可以试试将测试客户端归类好

### 启动服务端

回到`grpc-hello-world/`目录下，启动服务端`go run main.go server`，成功则仅返回
```
2018/02/26 17:19:36 gRPC and https listen on: 50052
```

### 执行测试客户端

回到`client`目录下，启动客户端`go run client.go`，成功则返回
```
2018/02/26 17:22:57 Grpc
```

### 执行测试Restful Api
```
curl -X POST -k https://localhost:50052/hello_world -d '{"referer": "restful_api"}'
```

成功则返回`{"message":"restful_api"}`

---

## 最终目录结构
```
grpc-hello-world
├── certs
│   ├── server.key
│   └── server.pem
├── client
│   └── client.go
├── cmd
│   ├── root.go
│   └── server.go
├── main.go
├── pkg
│   └── util
│       ├── grpc.go
│       └── tls.go
├── proto
│   ├── google
│   │   └── api
│   │       ├── annotations.pb.go
│   │       ├── annotations.proto
│   │       ├── http.pb.go
│   │       └── http.proto
│   ├── hello.pb.go
│   ├── hello.pb.gw.go
│   └── hello.proto
└── server
    ├── hello.go
    └── server.go
```

至此本节就结束了，推荐一下`jergoo`的文章，大家有时间可以看看

另外本节涉及了许多组件间的知识，值得大家细细的回味，非常有意义！

## 参考
### 示例代码
- [grpc-hello-world](https://github.com/EDDYCJY/grpc-hello-world)

