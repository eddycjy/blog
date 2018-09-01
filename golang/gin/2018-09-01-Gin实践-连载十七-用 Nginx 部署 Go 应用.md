# Golang Gin实践 连载十七 用 Nginx 部署 Go 应用

## 前言

如果已经看过前面 “十六部连载，两部番外”，相信您的能力已经有所提升

那么，现在今天来说说简单部署后端服务的事儿 🤓

## 做什么

在本章节，我们将简单介绍 Nginx 以及使用 Nginx 来完成对 [go-gin-example](https://github.com/EDDYCJY/go-gin-example) 的部署，会实现反向代理和简单负载均衡的功能


## Nginx

### 是什么

Nginx 是一个 Web Server，可以用作反向代理、负载均衡、邮件代理、TCP / UDP、HTTP 服务器等等，它拥有很多吸引人的特性，例如：

- 以较低的内存占用率处理 10,000 多个并发连接（每10k非活动HTTP保持活动连接约2.5 MB ）
- 静态服务器（处理静态文件）
- 正向、反向代理
- 负载均衡
- 通过OpenSSL 对 TLS / SSL 与 SNI 和 OCSP 支持
- FastCGI、SCGI、uWSGI 的支持
- WebSockets、HTTP/1.1 的支持
- Nginx + Lua

### 安装

请右拐谷歌或百度，安装好 Nginx 以备接下来的使用

### 简单讲解

#### 常用命令

- nginx：启动 Nginx
- nginx -s stop：立刻停止 Nginx 服务
- nginx -s reload：重新加载配置文件
- nginx -s quit：平滑停止 Nginx 服务
- nginx -t：测试配置文件是否正确
- nginx -v：显示 Nginx 版本信息
- nginx -V：显示 Nginx 版本信息、编译器和配置参数的信息

#### 涉及配置

1、 proxy_pass：配置**反向代理的路径**。需要注意的是如果 proxy_pass 的 url 最后为
/，则表示绝对路径。否则（不含变量下）表示相对路径，所有的路径都会被代理过去

2、 upstream：配置**负载均衡**，upstream 默认是以轮询的方式进行负载，另外还支持**四种模式**，分别是：

（1）weight：权重，指定轮询的概率，weight 与访问概率成正比

（2）ip_hash：按照访问 IP 的 hash 结果值分配

（3）fair：按后端服务器响应时间进行分配，响应时间越短优先级别越高

（4）url_hash：按照访问 URL 的 hash 结果值分配

## 部署

在这里需要对 nginx.conf 进行配置，如果你不知道对应的配置文件是哪个，可执行 `nginx -t` 看一下

```
$ nginx -t
nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful
```

显然，我的配置文件在 `/usr/local/etc/nginx/` 目录下，并且测试通过

### 反向代理

反向代理是指以代理服务器来接受网络上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。（来自[百科](https://baike.baidu.com/item/%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86/7793488?fr=aladdin)）

![image](https://i.imgur.com/Gx6ctv7.png)

#### 配置 hosts

由于需要用本机作为演示，因此先把映射配上去，打开 `/etc/hosts`，增加内容：

```
127.0.0.1       api.blog.com
```

#### 配置 nginx.conf

打开 nginx 的配置文件 nginx.conf（我的是 /usr/local/etc/nginx/nginx.conf），我们做了如下事情：

增加 server 片段的内容，设置 server_name 为 api.blog.com 并且监听 8081 端口，将所有路径转发到 `http://127.0.0.1:8000/` 下

```
worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       8081;
        server_name  api.blog.com;

        location / {
            proxy_pass http://127.0.0.1:8000/;
        }
    }
}
```

#### 验证

##### 启动 go-gin-example

回到 [go-gin-example](github.com/EDDYCJY/go-gin-example) 的项目下，执行 make，再运行 ./go-gin-exmaple

``` sh
$ make
github.com/EDDYCJY/go-gin-example
$ ls
LICENSE        README.md      conf           go-gin-example middleware     pkg            runtime        vendor
Makefile       README_ZH.md   docs           main.go        models         routers        service
$ ./go-gin-example 
...
[GIN-debug] DELETE /api/v1/articles/:id      --> github.com/EDDYCJY/go-gin-example/routers/api/v1.DeleteArticle (4 handlers)
[GIN-debug] POST   /api/v1/articles/poster/generate --> github.com/EDDYCJY/go-gin-example/routers/api/v1.GenerateArticlePoster (4 handlers)
Actual pid is 14672
```

##### 重启 nginx 

``` sh
$ nginx -t
nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful
$ nginx -s reload
```

##### 访问接口

![image](https://i.imgur.com/3AD99W4.jpg)

如此，就实现了一个简单的反向代理了，是不是很简单呢 

### 负载均衡

负载均衡，英文名称为Load Balance（常称 LB），其意思就是分摊到多个操作单元上进行执行（来自百科）

你能从运维口中经常听见，XXX 负载怎么突然那么高。 那么它到底是什么呢？

其背后一般有多台 server，系统会根据配置的策略（例如 Nginx 有提供四种选择）来进行动态调整，尽可能的达到各节点均衡，从而提高系统整体的吞吐量和快速响应

#### 如何演示

前提条件为多个后端服务，那么势必需要多个 [go-gin-example](https://github.com/EDDYCJY/go-gin-example)，为了演示我们可以启动多个端口，达到模拟的效果 

为了便于演示，分别在启动前将 conf/app.ini 的应用端口修改为 8001 和 8002（也可以做成传入参数的模式），达到启动 2 个监听 8001 和 8002 的后端服务

#### 配置 nginx.conf

回到 nginx.conf 的老地方，增加负载均衡所需的配置。新增 upstream 节点，设置其对应的 2 个后端服务，最后修改了 proxy_pass 指向（格式为 http:// + upstream 的节点名称）

```
worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    upstream api.blog.com {
        server 127.0.0.1:8001;
        server 127.0.0.1:8002;
    }

    server {
        listen       8081;
        server_name  api.blog.com;

        location / {
            proxy_pass http://api.blog.com/;
        }
    }
}
```

##### 重启 nginx 

``` sh
$ nginx -t
nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful
$ nginx -s reload
```

#### 验证

再重复访问 `http://api.blog.com:8081/auth?username={USER_NAME}}&password={PASSWORD}`，多访问几次便于查看效果

目前 Nginx 没有进行特殊配置，那么它是轮询策略，而 go-gin-example 默认开着 debug 模式，看看请求 log 就明白了

![image](https://i.imgur.com/L9IitGq.jpg)

![image](https://i.imgur.com/Bv5dCn0.jpg)

## 总结

在本章节，希望您能够简单习得日常使用的 Web Server 背后都是一些什么逻辑，Nginx 是什么？反向代理？负载均衡？

怎么简单部署，知道了吧。

## 参考
### 本系列示例代码
- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)
