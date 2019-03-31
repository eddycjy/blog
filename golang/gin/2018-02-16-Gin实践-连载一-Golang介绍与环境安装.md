# Golang介绍与环境安装

> Gin is a HTTP web framework written in Go (Golang). It features a Martini-like API with much better performance -- up to 40 times faster. If you need smashing performance, get yourself some Gin.

Gin是用Golang开发的一个微框架，类似Martinier的API，重点是小巧、易用、性能好很多，也因为 [httprouter](https://github.com/julienschmidt/httprouter) 的性能提高了40倍。

## 准备环节

### 一、安装Golang

首先，根据对应的操作系统选择安装包[下载](https://studygolang.com/dl)，

在这里我使用的是Centos 64位系统

``` sh
wget https://studygolang.com/dl/golang/go1.9.2.linux-amd64.tar.gz

tar -zxvf go1.9.2.linux-amd64.tar.gz

mv go/ /usr/local/
```

配置 /etc/profile

``` sh
vi /etc/profile
```
添加环境变量GOROOT和将GOBIN添加到PATH中

``` sh
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
```
添加环境变量GOPATH（这个可按实际情况设置目录位置）
``` sh
export GOPATH=/usr/local/go/path
```
配置完毕后，执行命令令其生效
``` sh
source /etc/profile
```

在控制台输入`go version`，若输出版本号则**安装成功**

那么大家会有些疑问，纠结`go`本身有什么东西，我们刚刚设置的环境变量是什么？

1、 `go`本身有什么东西

首先，我们在解压的时候会得到一个名为`go`的文件夹，其中包括了所有`Go`语言相关的一些文件，在这下面又包含很多文件夹和文件，我们来简单说明其中主要文件夹的作为：

- api：用于存放依照`Go`版本顺序的API增量列表文件。这里所说的API包含公开的变量、常量、函数等。这些API增量列表文件用于`Go`语言API检查
- bin：用于存放主要的标准命令文件（可执行文件），包含`go`、`godoc`、`gofmt`
- blog：用于存放官方博客中的所有文章
- doc：用于存放标准库的HTML格式的程序文档。我们可以通过`godoc`命令启动一个Web程序展示这些文档
- lib：用于存放一些特殊的库文件
- misc：用于存放一些辅助类的说明和工具
- pkg：用于存放安装`Go`标准库后的所有归档文件（以`.a`结尾的文件）。注意，你会发现其中有名称为`linux_amd64`的文件夹，我们称为平台相关目录。这类文件夹的名称由对应的操作系统和计算架构的名称组合而成。通过`go install`命令，`Go`程序会被编译成平台相关的归档文件存放到其中
- src：用于存放`Go`自身、`Go`标准工具以及标准库的所有源码文件
- test：存放用来测试和验证`Go`本身的所有相关文件

2、 刚刚设置的环境变量是什么
- GOROOT：`Go`的根目录
- GOPATH：用户工作区
- PATH下增加$GOROOT/bin：`Go`的`bin`下会存放可执行文件，我们把他加入PATH中就可以直接在命令行使用

3、 工作区是什么？

这在`Go`中是一个非常重要的概念，在一般情况下，`Go`源码文件必须放在工作区中，也就是说，我们写的项目代码都必须放在我们所设定的工作区中，虽然对于命令源码文件来说，这不是必须的。但我们大多都是前一种情况。工作区其实就是一个对应特定工程的目录，它应包含3个子目录：`src`目录、`pkg`目录、`bin`目录

- src：用于以代码包的形式组织并保存Go源码文件
- pkg：用于存放通过`go install`命令安装后的代码包的归档文件（.a 结尾的文件）
- bin：与pkg目录类似，在通过`go install`命令完成安装后，保存由Go命令源码文件生成的可执行文件

4、 什么是命令源码文件？

如果一个源码文件被声明属于`main`代码包，且该文件代码中包含无参数声明和结果声明的`main`函数，则它就是命令源码文件。命令源码文件可通过`go run`命令直接启动运行

### 二、安装Govendor
> If using go1.5, ensure GO15VENDOREXPERIMENT=1 is set.

在命令行下执行安装
``` sh
go get -u github.com/kardianos/govendor
```
等待一会，安装成功后。

我们`cd  /usr/local/go/path`（第三方依赖包，会默认安装在GOPATH的第一个目录下）目录，

执行`ls`，可以在工作区中看到`bin`、`pkg`、`src`三个目录。这就是我们上面一小节所说**的工作区**了！

那么，我们所安装的govendor去哪里了呢？


答案就在工作区内，所生成的代码包大概是这样。我们所需要的是编译好的可执行文件，在`/usr/local/go/path/bin`中。
``` sh
path/
├── bin
│   └── govendor
├── pkg
│   └── linux_amd64
│       └── github.com
│           └── kardianos
│               └── govendor
│                   ├── ...
└── src
    └── github.com
        └── kardianos
            └── govendor
                ├── ...
```

大家还记得我们先前在环境变量`PATH`中设置了GOBIN，

我们现在要做的就是把工作区中`bin`目录下的可执行文件`govendor`给移动过去，或者你可以将$GOPATH的BIN目录给加入环境变量中

那样就可以直接在命令行直接执行`govendor`了

``` sh
mv /usr/local/go/path/bin/govendor /usr/local/go/bin/
```

移动成功后，在命令行执行`govendor -version`，若出现版本号，则成功

``` sh
#govendor -version
$ v1.0.9
```

在这里为什么单独挑出一节来讲`govendor`呢？

大家可以想想，虽然我们在本地开发，利用`$GOPATH`达到安装第三方依赖包，然后去使用他，似乎也没有什么问题。

但是在实际的多人协作及部署中是有问题的
- 每一个新来的人都要`go get`很多次
- 拉下来的版本还可能不一样
- 线上部署更麻烦了

因此我们在这简单的使用`govendor`来解决这个问题，在这个项目完成的最后，你只需`govendor init`再`govendor add +external`就能美滋滋的把依赖包都放到项目的`vendor`目录中，就能把它一同传上你的版本库里了，是不是很方便呢。

当然了，目前官方推荐的包管理工具就有十几种，大家可以适当考察一下，这个不在本篇的范围内。

### 三、安装Gin
在命令行下执行安装
``` sh
go get -u github.com/gin-gonic/gin
```

检查`/usr/local/go/path`中是否存在`gin`的代码包

### 四、测试Gin是否安装成功
编写一个`test.go`文件

``` go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080
}
```

执行`test.go`
``` sh
go run test.go
```

访问$HOST:8080/ping，若返回`{"message":"pong"}`则正确
``` sh
curl 127.0.0.1:8080/ping
```

至此，我们的环境安装都基本完成了:)

## 参考
### 本系列示例代码
- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

### 相关文档
- [Gin](https://github.com/gin-gonic/gin)
- [Gin Web Framework](https://gin-gonic.github.io/gin/)
- Go并发编程实战
- [govendor](https://github.com/kardianos/govendor)



