---

title:      "用 Go 来了解一下 Redis 通讯协议"
date:       2018-06-07 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
---

Go、PHP、Java... 都有那么多包来支撑你使用 Redis，那你是否有想过

有了服务端，有了客户端，他们俩是怎样通讯，又是基于什么通讯协议做出交互的呢？

## 介绍

基于我们的目的，本文主要讲解和实践 Redis 的通讯协议

Redis 的客户端和服务端是通过 TCP 连接来进行数据交互， 服务器默认的端口号为 6379

客户端和服务器发送的命令或数据一律以 \r\n （CRLF）结尾（这是一条约定）

## 协议

在 Redis 中分为**请求**和**回复**，而请求协议又分为新版和旧版，新版统一请求协议在 Redis 1.2 版本中引入，最终在 Redis 2.0 版本成为 Redis 服务器通信的标准方式

本文是基于新版协议来实现功能，不建议使用旧版（1.2 挺老旧了）。如下是新协议的各种范例：

### 请求协议

1、 格式示例

```
*<参数数量> CR LF
$<参数 1 的字节数量> CR LF
<参数 1 的数据> CR LF
...
$<参数 N 的字节数量> CR LF
<参数 N 的数据> CR LF
```

在该协议下所有发送至 Redis 服务器的参数都是二进制安全（binary safe）的

2、打印示例

```
*3
$3
SET
$5
mykey
$7
myvalue
```

3、实际协议值

```
"*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$7\r\nmyvalue\r\n"
```

这就是 Redis 的请求协议规范，按照范例1编写客户端逻辑，最终发送的是范例3，相信你已经有大致的概念了，Redis 的协议非常的简洁易懂，这也是好上手的原因之一，你可以想想协议这么定义的好处在哪？

### 回复

Redis 会根据你请求协议的不同（执行的命令结果也不同），返回多种不同类型的回复。在这个回复“协议”中，可以通过检查第一个字节，确定这个回复是什么类型，如下：

- 状态回复（status reply）的第一个字节是 "+"
- 错误回复（error reply）的第一个字节是 "-"
- 整数回复（integer reply）的第一个字节是 ":"
- 批量回复（bulk reply）的第一个字节是 "$"
- 多条批量回复（multi bulk reply）的第一个字节是 "*"


有了回复的头部标识，结尾的 CRLF，你可以大致猜想出回复“协议”是怎么样的，但是实践才能得出真理，斎知道怕是你很快就忘记了 😀


## 实践

### 与 Redis 服务器交互

```
package main

import (
	"log"
	"net"
	"os"

	"github.com/EDDYCJY/redis-protocol-example/protocol"
)

const (
	Address = "127.0.0.1:6379"
	Network = "tcp"
)

func Conn(network, address string) (net.Conn, error) {
	conn, err := net.Dial(network, address)
	if err != nil {
		return nil, err
	}

	return conn, nil
}

func main() {
        // 读取入参
	args := os.Args[1:]
	if len(args) <= 0 {
		log.Fatalf("Os.Args <= 0")
	}
    
        // 获取请求协议
	reqCommand := protocol.GetRequest(args)
	
	// 连接 Redis 服务器
	redisConn, err := Conn(Network, Address)
	if err != nil {
		log.Fatalf("Conn err: %v", err)
	}
	defer redisConn.Close()
    
        // 写入请求内容
	_, err = redisConn.Write(reqCommand)
	if err != nil {
		log.Fatalf("Conn Write err: %v", err)
	}
    
        // 读取回复
	command := make([]byte, 1024)
	n, err := redisConn.Read(command)
	if err != nil {
		log.Fatalf("Conn Read err: %v", err)
	}
    
        // 处理回复
	reply, err := protocol.GetReply(command[:n])
	if err != nil {
		log.Fatalf("protocol.GetReply err: %v", err)
	}
    
        // 处理后的回复内容
	log.Printf("Reply: %v", reply)
	// 原始的回复内容
	log.Printf("Command: %v", string(command[:n]))
}
```

在这里我们完成了整个 Redis 客户端和服务端交互的流程，分别如下：

1、读取命令行参数：获取执行的 Redis 命令

2、获取请求协议参数

3、连接 Redis 服务器，获取连接句柄

4、将请求协议参数写入连接：发送请求的命令行参数

5、从连接中读取返回的数据：读取先前请求的回复数据

6、根据回复“协议”内容，处理回复的数据集

7、输出处理后的回复内容及原始回复内容

### 请求

```
func GetRequest(args []string) []byte {
	req := []string{
		"*" + strconv.Itoa(len(args)),
	}

	for _, arg := range args {
		req = append(req, "$"+strconv.Itoa(len(arg)))
		req = append(req, arg)
	}

	str := strings.Join(req, "\r\n")
	return []byte(str + "\r\n")
}
```

通过对 Redis 的请求协议的分析，可得出它的规律，先加上标志位，计算参数总数量，再循环合并各个参数的字节数量、值就可以了

### 回复

```
func GetReply(reply []byte) (interface{}, error) {
	replyType := reply[0]
	switch replyType {
	case StatusReply:
		return doStatusReply(reply[1:])
	case ErrorReply:
		return doErrorReply(reply[1:])
	case IntegerReply:
		return doIntegerReply(reply[1:])
	case BulkReply:
		return doBulkReply(reply[1:])
	case MultiBulkReply:
		return doMultiBulkReply(reply[1:])
	default:
		return nil, nil
	}
}

func doStatusReply(reply []byte) (string, error) {
	if len(reply) == 3 && reply[1] == 'O' && reply[2] == 'K' {
		return OkReply, nil
	}

	if len(reply) == 5 && reply[1] == 'P' && reply[2] == 'O' && reply[3] == 'N' && reply[4] == 'G' {
		return PongReply, nil
	}

	return string(reply), nil
}

func doErrorReply(reply []byte) (string, error) {
	return string(reply), nil
}

func doIntegerReply(reply []byte) (int, error) {
	pos := getFlagPos('\r', reply)
	result, err := strconv.Atoi(string(reply[:pos]))
	if err != nil {
		return 0, err
	}

	return result, nil
}

...
```

在这里我们对所有回复类型进行了分发，不同的回复标志位对应不同的处理方式，在这里需求注意几项问题，如下：

1、当请求的值不存在，会将特殊值 -1 用作回复

2、服务器发送的所有字符串都由 CRLF 结尾

3、多条批量回复是可基于批量回复的，要注意理解

4、无内容的多条批量回复是存在的

最重要的是，对不同回复的规则的把控，能够让你更好的理解 Redis 的请求、回复的交互过程 👌

## 小结

写这篇文章的起因，是因为常常在使用 Redis 时，只是用，你不知道它是基于什么样的通讯协议来通讯，这样的感觉是十分难受的

通过本文的讲解，我相信你已经大致了解 Redis 客户端是怎么样和服务端交互，也清楚了其所用的通讯原理，希望能够对你有所帮助！

最后，如果想详细查看代码，右拐项目地址：https://github.com/EDDYCJY/redis-protocol-example

如果对你有所帮助，欢迎点个 Star 👍

## 参考

- [通信协议](http://doc.redisfans.com/topic/protocol.html)
