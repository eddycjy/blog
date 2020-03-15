---

title:      "「连载九」将Golang应用部署到Docker"
date:       2018-03-24 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - gin
---

## 涉及知识点

- Go + Docker

## 本文目标

将我们的 `go-gin-example` 应用部署到一个 Docker 里，你需要先准备好如下东西：

- 你需要安装好 `docker`。
- 如果上外网比较吃力，需要配好镜像源。

## Docker

在这里简单介绍下 Docker，建议深入学习

![image](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1521800047226&di=28b2764fccca8a943aea7d79ad8aed98&imgtype=0&src=http%3A%2F%2Fwww.cww.net.cn%2FupLoadFile%2F2014%2F6%2F13%2F201461382247734.png)

Docker 是一个开源的轻量级容器技术，让开发者可以打包他们的应用以及应用运行的上下文环境到一个可移植的镜像中，然后发布到任何支持 Docker 的系统上运行。 通过容器技术，在几乎没有性能开销的情况下，Docker 为应用提供了一个隔离运行环境

- 简化配置
- 代码流水线管理
- 提高开发效率
- 隔离应用
- 快速、持续部署

---

接下来我们正式开始对项目进行 `docker` 的所需处理和编写，每一个大标题为步骤大纲

## Golang

### 一、编写 Dockerfile

在 `go-gin-example` 项目根目录创建 Dockerfile 文件，写入内容

```
FROM golang:latest

ENV GOPROXY https://goproxy.cn,direct
WORKDIR $GOPATH/src/github.com/EDDYCJY/go-gin-example
COPY . $GOPATH/src/github.com/EDDYCJY/go-gin-example
RUN go build .

EXPOSE 8000
ENTRYPOINT ["./go-gin-example"]
```

#### 作用

`golang:latest` 镜像为基础镜像，将工作目录设置为 `$GOPATH/src/go-gin-example`，并将当前上下文目录的内容复制到 `$GOPATH/src/go-gin-example` 中

在进行 `go build` 编译完毕后，将容器启动程序设置为 `./go-gin-example`，也就是我们所编译的可执行文件

注意 `go-gin-example` 在 `docker` 容器里编译，并没有在宿主机现场编译

#### 说明

Dockerfile 文件是用于定义 Docker 镜像生成流程的配置文件，文件内容是一条条指令，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建；这些指令应用于基础镜像并最终创建一个新的镜像

你可以认为用于快速创建自定义的 Docker 镜像

**1、 FROM**

指定基础镜像（必须有的指令，并且必须是第一条指令）

**2、 WORKDIR**

格式为 `WORKDIR` <工作目录路径>

使用 `WORKDIR` 指令可以来**指定工作目录**（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如果目录不存在，`WORKDIR` 会帮你建立目录

**3、COPY**

格式：

    COPY <源路径>... <目标路径>
    COPY ["<源路径1>",... "<目标路径>"]

`COPY` 指令将从构建上下文目录中 <源路径> 的文件/目录**复制**到新的一层的镜像内的 <目标路径> 位置

**4、RUN**

用于执行命令行命令

格式：`RUN` <命令>

**5、EXPOSE**

格式为 `EXPOSE` <端口 1> [<端口 2>...]

`EXPOSE` 指令是**声明运行时容器提供服务端口，这只是一个声明**，在运行时并不会因为这个声明应用就会开启这个端口的服务

在 Dockerfile 中写入这样的声明有两个好处

- 帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射
- 运行时使用随机端口映射时，也就是 `docker run -P` 时，会自动随机映射 `EXPOSE` 的端口

**6、ENTRYPOINT**

`ENTRYPOINT` 的格式和 `RUN` 指令格式一样，分为两种格式

- `exec` 格式：

```
<ENTRYPOINT> "<CMD>"
```

- `shell` 格式：

```
ENTRYPOINT [ "curl", "-s", "http://ip.cn" ]
```

`ENTRYPOINT` 指令是**指定容器启动程序及参数**

### 二、构建镜像

`go-gin-example` 的项目根目录下**执行** `docker build -t gin-blog-docker .`

该命令作用是创建/构建镜像，`-t` 指定名称为 `gin-blog-docker`，`.` 构建内容为当前上下文目录

```
$ docker build -t gin-blog-docker .
Sending build context to Docker daemon 96.39 MB
Step 1/6 : FROM golang:latest
 ---> d632bbfe5767
Step 2/6 : WORKDIR $GOPATH/src/github.com/EDDYCJY/go-gin-example
 ---> 56294f978c5d
Removing intermediate container e112997b995d
Step 3/6 : COPY . $GOPATH/src/github.com/EDDYCJY/go-gin-example
 ---> 3b60960120cf
Removing intermediate container 63e310b3f60c
Step 4/6 : RUN go build .
 ---> Running in 52648a431450
go: downloading github.com/gin-gonic/gin v1.3.0
go: downloading github.com/go-ini/ini v1.32.1-0.20180214101753-32e4be5f41bb
go: downloading github.com/swaggo/gin-swagger v1.0.1-0.20190110070702-0c6fcfd3c7f3
...
 ---> 7bfbeb301fea
Removing intermediate container 52648a431450
Step 5/6 : EXPOSE 8000
 ---> Running in 98f5b387d1bb
 ---> b65bd4076c65
Removing intermediate container 98f5b387d1bb
Step 6/6 : ENTRYPOINT ./go-gin-example
 ---> Running in c4f6cdeb667b
 ---> d8a109c7697c
Removing intermediate container c4f6cdeb667b
Successfully built d8a109c7697c
```

### 三、验证镜像

查看所有的镜像，确定刚刚构建的 `gin-blog-docker` 镜像是否存在

```
$ docker images
REPOSITORY              TAG                 IMAGE ID            CREATED              SIZE
gin-blog-docker         latest              d8a109c7697c        About a minute ago   946 MB
docker.io/golang        latest              d632bbfe5767        8 days ago           779 MB
...
```

### 四、创建并运行一个新容器

执行命令 `docker run -p 8000:8000 gin-blog-docker`

```
$ docker run -p 8000:8000 gin-blog-docker
dial tcp 127.0.0.1:3306: connect: connection refused
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

...
Actual pid is 1

```

运行成功，你以为大功告成了吗？

你想太多了，仔细看看控制台的输出了一条错误 `dial tcp 127.0.0.1:3306: connect: connection refused`

我们研判一下，发现是 `Mysql` 的问题，接下来第二项我们将解决这个问题

## Mysql

### 一、拉取镜像

从 `Docker` 的公共仓库 `Dockerhub` 下载 `MySQL` 镜像（国内建议配个镜像）

```
$ docker pull mysql
```

### 二、创建并运行一个新容器

运行 `Mysql` 容器，并设置执行成功后返回容器 ID

```
$ docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=rootroot -d mysql
8c86ac986da4922492934b6fe074254c9165b8ee3e184d29865921b0fef29e64
```

#### 连接 Mysql

初始化的 `Mysql` 应该如图

![image](https://i.loli.net/2018/03/23/5ab4caab04cf1.png)

## Golang + Mysql

### 一、删除镜像

由于原本的镜像存在问题，我们需要删除它，此处有几种做法

- 删除原本有问题的镜像，重新构建一个新镜像
- 重新构建一个不同 `name`、`tag` 的新镜像

删除原本的有问题的镜像，`-f` 是强制删除及其关联状态

若不执行 `-f`，你需要执行 `docker ps -a` 查到所关联的容器，将其 `rm` 解除两者依赖关系

```
$ docker rmi -f gin-blog-docker
Untagged: gin-blog-docker:latest
Deleted: sha256:d8a109c7697c3c2d9b4de7dbb49669d10106902122817b6467a031706bc52ab4
Deleted: sha256:b65bd4076c65a3c24029ca4def3b3f37001ff7c9eca09e2590c4d29e1e23dce5
Deleted: sha256:7bfbeb301fea9d8912a4b7c43e4bb8b69bdc57f0b416b372bfb6510e476a7dee
Deleted: sha256:3b60960120cf619181c1762cdc1b8ce318b8c815e056659809252dd321bcb642
Deleted: sha256:56294f978c5dfcfa4afa8ad033fd76b755b7ecb5237c6829550741a4d2ce10bc
```

### 二、修改配置文件

将项目的配置文件 `conf/app.ini`，内容修改为

```ini
#debug or release
RUN_MODE = debug

[app]
PAGE_SIZE = 10
JWT_SECRET = 233

[server]
HTTP_PORT = 8000
READ_TIMEOUT = 60
WRITE_TIMEOUT = 60

[database]
TYPE = mysql
USER = root
PASSWORD = rootroot
HOST = mysql:3306
NAME = blog
TABLE_PREFIX = blog_

```

### 三、重新构建镜像

重复先前的步骤，回到 `gin-blog` 的项目根目录下**执行** `docker build -t gin-blog-docker .`

### 四、创建并运行一个新容器

## 关联

Q：我们需要将 `Golang` 容器和 `Mysql` 容器关联起来，那么我们需要怎么做呢？

A：增加命令 `--link mysql:mysql` 让 `Golang` 容器与 `Mysql` 容器互联；通过 `--link`，**可以在容器内直接使用其关联的容器别名进行访问**，而不通过 IP，但是`--link`只能解决单机容器间的关联，在分布式多机的情况下，需要通过别的方式进行连接

## 运行

执行命令 `docker run --link mysql:mysql -p 8000:8000 gin-blog-docker`

```
$ docker run --link mysql:mysql -p 8000:8000 gin-blog-docker
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)
...
Actual pid is 1
```

## 结果

检查启动输出、接口测试、数据库内数据，均正常；我们的 `Golang` 容器和 `Mysql` 容器成功关联运行，大功告成 :)

---

## Review

### 思考

虽然应用已经能够跑起来了

但如果对 `Golang` 和 `Docker` 有一定的了解，我希望你能够想到至少 2 个问题

- 为什么 `gin-blog-docker` 占用空间这么大？（可用 `docker ps -as | grep gin-blog-docker` 查看）
- `Mysql` 容器直接这么使用，数据存储到哪里去了？

### 创建超小的 Golang 镜像

Q：第一个问题，为什么这么镜像体积这么大？

A：`FROM golang:latest` 拉取的是官方 `golang` 镜像，包含 Golang 的编译和运行环境，外加一堆 GCC、build 工具，相当齐全

这是有问题的，**我们可以不在 Golang 容器中现场编译的**，压根用不到那些东西，我们只需要一个能够运行可执行文件的环境即可

#### 构建 Scratch 镜像

Scratch 镜像，简洁、小巧，基本是个空镜像

##### 一、修改 Dockerfile

```
FROM scratch

WORKDIR $GOPATH/src/github.com/EDDYCJY/go-gin-example
COPY . $GOPATH/src/github.com/EDDYCJY/go-gin-example

EXPOSE 8000
CMD ["./go-gin-example"]
```

##### 二、编译可执行文件

```
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o go-gin-example .
```

编译所生成的可执行文件会依赖一些库，并且是动态链接。在这里因为使用的是 `scratch` 镜像，它是空镜像，因此我们需要将生成的可执行文件静态链接所依赖的库

##### 三、构建镜像

```
$ docker build -t gin-blog-docker-scratch .
Sending build context to Docker daemon 133.1 MB
Step 1/5 : FROM scratch
 --->
Step 2/5 : WORKDIR $GOPATH/src/github.com/EDDYCJY/go-gin-example
 ---> Using cache
 ---> ee07e166a638
Step 3/5 : COPY . $GOPATH/src/github.com/EDDYCJY/go-gin-example
 ---> 1489a0693d51
Removing intermediate container e3e9efc0fe4d
Step 4/5 : EXPOSE 8000
 ---> Running in b5630de5544a
 ---> 6993e9f8c944
Removing intermediate container b5630de5544a
Step 5/5 : CMD ./go-gin-example
 ---> Running in eebc0d8628ae
 ---> 5310bebeb86a
Removing intermediate container eebc0d8628ae
Successfully built 5310bebeb86a
```

注意，假设你的 Golang 应用没有依赖任何的配置等文件，是可以直接把可执行文件给拷贝进去即可，其他都不必关心

这里可以有好几种解决方案

- 依赖文件统一管理挂载
- go-bindata 一下

...

因此这里如果**解决了文件依赖的问题**后，就不需要把目录给 `COPY` 进去了

##### 四、运行

```
$ docker run --link mysql:mysql -p 8000:8000 gin-blog-docker-scratch
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /auth                     --> github.com/EDDYCJY/go-gin-example/routers/api.GetAuth (3 handlers)
...
```

成功运行，程序也正常接收请求

接下来我们再看看占用大小，执行 `docker ps -as` 命令

```
$ docker ps -as
CONTAINER ID        IMAGE                     COMMAND                  ...         SIZE
9ebdba5a8445        gin-blog-docker-scratch   "./go-gin-example"       ...     0 B (virtual 132 MB)
427ee79e6857        gin-blog-docker           "./go-gin-example"       ...     0 B (virtual 946 MB)
```

从结果而言，占用大小以`Scratch`镜像为基础的容器完胜，完成目标

### Mysql 挂载数据卷

倘若不做任何干涉，在每次启动一个 `Mysql` 容器时，数据库都是空的。另外容器删除之后，数据就丢失了（还有各类意外情况），非常糟糕！

#### 数据卷

数据卷 是被设计用来持久化数据的，它的生命周期独立于容器，Docker 不会在容器被删除后自动删除 数据卷，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的 数据卷。如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 `docker rm -v` 这个命令

数据卷 是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：

- 数据卷 可以在容器之间共享和重用

- 对 数据卷 的修改会立马生效

- 对 数据卷 的更新，不会影响镜像

- 数据卷 默认会一直存在，即使容器被删除

> 注意：数据卷 的使用，类似于 Linux 下对目录或文件进行 mount，镜像中的被指定为挂载点的目录中的文件会隐藏掉，能显示看的是挂载的 数据卷。

#### 如何挂载

首先创建一个目录用于存放数据卷；示例目录 `/data/docker-mysql`，注意 `--name` 原本名称为 `mysql` 的容器，需要将其删除 `docker rm`

```
$ docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=rootroot -v /data/docker-mysql:/var/lib/mysql -d mysql
54611dbcd62eca33fb320f3f624c7941f15697d998f40b24ee535a1acf93ae72
```

创建成功，检查目录 `/data/docker-mysql`，下面多了不少数据库文件

#### 验证

接下来交由你进行验证，目标是创建一些测试表和数据，然后删除当前容器，重新创建的容器，数据库数据也依然存在（当然了数据卷指向要一致）

我已验证完毕，你呢？

## 参考

### 本系列示例代码

- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

### 书籍

- [Docker —— 从入门到实践](https://www.gitbook.com/book/yeasy/docker_practice/details)

## 关于

### 修改记录

- 第一版：2018 年 02 月 16 日发布文章
- 第二版：2019 年 10 月 01 日修改文章

## ？

如果有任何疑问或错误，欢迎在 [issues](https://github.com/EDDYCJY/blog) 进行提问或给予修正意见，如果喜欢或对你有所帮助，欢迎 Star，对作者是一种鼓励和推进。

### 我的公众号

![image](https://image.eddycjy.com/8d0b0c3a11e74efd5fdfd7910257e70b.jpg)