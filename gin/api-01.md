# Gin搭建Blog API's （一）

项目地址：https://github.com/EDDYCJY/go-gin-example

## 思考

首先，在一个初始项目开始前，大家都要思考一下

- 程序的文本配置写在代码中，好吗？

- API 的错误码硬编码在程序中，合适吗？

- db句柄谁都去`Open`，没有统一管理，好吗？

- 获取分页等公共参数，谁都自己写一套逻辑，好吗？

显然在较正规的项目中，这些问题的答案都是**不可以**，为了解决这些问题，我们挑选一款读写配置文件的库，目前比较火的有 [viper](https://github.com/spf13/viper)，有兴趣你未来可以简单了解一下，没兴趣的话等以后接触到再说。

但是本系列选用 [go-ini/ini](https://github.com/go-ini/ini) ，它的 [中文文档](https://ini.unknwon.io/)。大家是必须需要要简单阅读它的文档，再接着完成后面的内容。

## 本文目标

- 编写一个简单的API错误码包。
- 完成一个 Demo 示例。
- 讲解 Demo 所涉及的知识点。

## 介绍和初始化项目

### 初始化项目目录

在前一章节中，我们初始化了一个 `go-gin-example` 项目，接下来我们需要继续新增如下目录结构：

``` sh
go-gin-example/
├── conf
├── middleware
├── models
├── pkg
├── routers
└── runtime
```

- conf：用于存储配置文件
- middleware：应用中间件
- models：应用数据库模型
- pkg：第三方包
- routers 路由逻辑处理
- runtime：应用运行时数据

### 初始项目数据库

新建 `blog` 数据库，编码为`utf8_general_ci`，在 `blog` 数据库下，新建以下表

**1、 标签表**

```
CREATE TABLE `blog_tag` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(100) DEFAULT '' COMMENT '标签名称',
  `created_on` int(10) unsigned DEFAULT '0' COMMENT '创建时间',
  `created_by` varchar(100) DEFAULT '' COMMENT '创建人',
  `modified_on` int(10) unsigned DEFAULT '0' COMMENT '修改时间',
  `modified_by` varchar(100) DEFAULT '' COMMENT '修改人',
  `deleted_on` int(10) unsigned DEFAULT '0',
  `state` tinyint(3) unsigned DEFAULT '1' COMMENT '状态 0为禁用、1为启用',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='文章标签管理';
```

**2、 文章表**

```
CREATE TABLE `blog_article` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `tag_id` int(10) unsigned DEFAULT '0' COMMENT '标签ID',
  `title` varchar(100) DEFAULT '' COMMENT '文章标题',
  `desc` varchar(255) DEFAULT '' COMMENT '简述',
  `content` text,
  `created_on` int(11) DEFAULT NULL,
  `created_by` varchar(100) DEFAULT '' COMMENT '创建人',
  `modified_on` int(10) unsigned DEFAULT '0' COMMENT '修改时间',
  `modified_by` varchar(255) DEFAULT '' COMMENT '修改人',
  `deleted_on` int(10) unsigned DEFAULT '0',
  `state` tinyint(3) unsigned DEFAULT '1' COMMENT '状态 0为禁用1为启用',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='文章管理';
```

**3、 认证表**

```
CREATE TABLE `blog_auth` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `username` varchar(50) DEFAULT '' COMMENT '账号',
  `password` varchar(50) DEFAULT '' COMMENT '密码',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `blog`.`blog_auth` (`id`, `username`, `password`) VALUES (null, 'test', 'test123456');

```

## 编写项目配置包

在 `go-gin-example` 应用目录下，拉取 `go-ini/ini` 的依赖包，如下：

```
$ go get -u github.com/go-ini/ini
go: finding github.com/go-ini/ini v1.48.0
go: downloading github.com/go-ini/ini v1.48.0
go: extracting github.com/go-ini/ini v1.48.0
```
接下来我们需要编写基础的应用配置文件，在 `go-gin-example` 的`conf`目录下新建`app.ini`文件，写入内容：
```
#debug or release
RUN_MODE = debug

[app]
PAGE_SIZE = 10
JWT_SECRET = 23347$040412

[server]
HTTP_PORT = 8000
READ_TIMEOUT = 60
WRITE_TIMEOUT = 60

[database]
TYPE = mysql
USER = 数据库账号
PASSWORD = 数据库密码
#127.0.0.1:3306
HOST = 数据库IP:数据库端口号
NAME = blog
TABLE_PREFIX = blog_
```

建立调用配置的`setting`模块，在`go-gin-example`的`pkg`目录下新建`setting`目录，新建 `setting.go` 文件，写入内容：
```
package setting

import (
	"log"
	"time"

	"github.com/go-ini/ini"
)

var (
	Cfg *ini.File

	RunMode string
	
	HTTPPort int
	ReadTimeout time.Duration
	WriteTimeout time.Duration

	PageSize int
	JwtSecret string
)

func init() {
	var err error
	Cfg, err = ini.Load("conf/app.ini")
	if err != nil {
		log.Fatalf("Fail to parse 'conf/app.ini': %v", err)
	}

	LoadBase()
	LoadServer()
	LoadApp()
}

func LoadBase() {
	RunMode = Cfg.Section("").Key("RUN_MODE").MustString("debug")
}

func LoadServer() {
	sec, err := Cfg.GetSection("server")
	if err != nil {
		log.Fatalf("Fail to get section 'server': %v", err)
	}

	HTTPPort = sec.Key("HTTP_PORT").MustInt(8000)
	ReadTimeout = time.Duration(sec.Key("READ_TIMEOUT").MustInt(60)) * time.Second
	WriteTimeout =  time.Duration(sec.Key("WRITE_TIMEOUT").MustInt(60)) * time.Second	
}

func LoadApp() {
	sec, err := Cfg.GetSection("app")
	if err != nil {
		log.Fatalf("Fail to get section 'app': %v", err)
	}

	JwtSecret = sec.Key("JWT_SECRET").MustString("!@)*#)!@U#@*!@!)")
	PageSize = sec.Key("PAGE_SIZE").MustInt(10)
}
```

当前的目录结构：
```
go-gin-example
├── conf
│   └── app.ini
├── go.mod
├── go.sum
├── middleware
├── models
├── pkg
│   └── setting.go
├── routers
└── runtime
```

## 编写API错误码包
建立错误码的`e`模块，在`go-gin-example`的`pkg`目录下新建`e`目录，新建`code.go`和`msg.go`文件，写入内容：

**1、 code.go：**

```
package e

const (
	SUCCESS = 200
	ERROR = 500
	INVALID_PARAMS = 400

	ERROR_EXIST_TAG = 10001
	ERROR_NOT_EXIST_TAG = 10002
	ERROR_NOT_EXIST_ARTICLE = 10003

	ERROR_AUTH_CHECK_TOKEN_FAIL = 20001
	ERROR_AUTH_CHECK_TOKEN_TIMEOUT = 20002
	ERROR_AUTH_TOKEN = 20003
	ERROR_AUTH = 20004
)
```

**2、 msg.go：**

```
package e

var MsgFlags = map[int]string {
	SUCCESS : "ok",
	ERROR : "fail",
	INVALID_PARAMS : "请求参数错误",
	ERROR_EXIST_TAG : "已存在该标签名称",
	ERROR_NOT_EXIST_TAG : "该标签不存在",
	ERROR_NOT_EXIST_ARTICLE : "该文章不存在",
	ERROR_AUTH_CHECK_TOKEN_FAIL : "Token鉴权失败",
	ERROR_AUTH_CHECK_TOKEN_TIMEOUT : "Token已超时",
	ERROR_AUTH_TOKEN : "Token生成失败",
	ERROR_AUTH : "Token错误",
}

func GetMsg(code int) string {
	msg, ok := MsgFlags[code]
	if ok {
		return msg
	}

	return MsgFlags[ERROR]
}
```

## 编写工具包
在`go-gin-example`的`pkg`目录下新建`util`目录，并拉取`com`的依赖包，如下：

```
go get -u github.com/unknwon/com
```

### 编写分页页码的获取方法
在`util`目录下新建`pagination.go`，写入内容：
```
package util

import (
	"github.com/gin-gonic/gin"
	"github.com/unknwon/com"

	"github.com/EDDYCJY/go-gin-example/pkg/setting"
)

func GetPage(c *gin.Context) int {
	result := 0
	page, _ := com.StrTo(c.Query("page")).Int()
    if page > 0 {
        result = (page - 1) * setting.PageSize
    }

    return result
}
```

## 编写models init
拉取`gorm`的依赖包，如下：
```
go get -u github.com/jinzhu/gorm
```
拉取`mysql`驱动的依赖包，如下：
```
go get -u github.com/go-sql-driver/mysql
```
完成后，在`go-gin-example`的`models`目录下新建`models.go`，用于`models`的初始化使用
```
package models

import (
	"log"
	"fmt"

	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/mysql"

	"github.com/EDDYCJY/go-gin-example/pkg/setting"
)

var db *gorm.DB

type Model struct {
	ID int `gorm:"primary_key" json:"id"`
	CreatedOn int `json:"created_on"`
	ModifiedOn int `json:"modified_on"`
}

func init() {
	var (
		err error
		dbType, dbName, user, password, host, tablePrefix string
	)

	sec, err := setting.Cfg.GetSection("database")
	if err != nil {
		log.Fatal(2, "Fail to get section 'database': %v", err)
	}

	dbType = sec.Key("TYPE").String()
	dbName = sec.Key("NAME").String()
	user = sec.Key("USER").String()
	password = sec.Key("PASSWORD").String()
	host = sec.Key("HOST").String()
	tablePrefix = sec.Key("TABLE_PREFIX").String()

	db, err = gorm.Open(dbType, fmt.Sprintf("%s:%s@tcp(%s)/%s?charset=utf8&parseTime=True&loc=Local", 
		user, 
		password, 
		host, 
		dbName))

	if err != nil {
		log.Println(err)
	}

	gorm.DefaultTableNameHandler = func (db *gorm.DB, defaultTableName string) string  {
	    return tablePrefix + defaultTableName;
	}

	db.SingularTable(true)
	db.LogMode(true)
	db.DB().SetMaxIdleConns(10)
	db.DB().SetMaxOpenConns(100)
}

func CloseDB() {
	defer db.Close()
}
```

## 编写项目启动、路由文件

最基础的准备工作完成啦，让我们开始编写Demo吧！

### 编写Demo

在`go-gin-example`下建立`main.go`作为启动文件（也就是`main`包），我们先写个**Demo**，帮助大家理解，写入文件内容：

```
package main

import (
    "fmt"
	  "net/http"

    "github.com/gin-gonic/gin"

	  "github.com/EDDYCJY/go-gin-example/pkg/setting"
)

func main() {
	router := gin.Default()
    router.GET("/test", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "test",
		})
	})

	s := &http.Server{
		Addr:           fmt.Sprintf(":%d", setting.HTTPPort),
		Handler:        router,
		ReadTimeout:    setting.ReadTimeout,
		WriteTimeout:   setting.WriteTimeout,
		MaxHeaderBytes: 1 << 20,
	}

	s.ListenAndServe()
}
```

执行`go run main.go`，查看命令行是否显示
```
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /test                     --> main.main.func1 (3 handlers)
```

在本机执行`curl 127.0.0.1:8000/test`，检查是否返回`{"message":"test"}`。

### 知识点

**那么，我们来延伸一下Demo所涉及的知识点！**

##### 标准库

- [fmt](https://golang.org/pkg/fmt/)：实现了类似C语言printf和scanf的格式化I/O。格式化动作（'verb'）源自C语言但更简单
- [net/http](https://golang.org/pkg/net/http/)：提供了HTTP客户端和服务端的实现

##### **Gin**

- [gin.Default()](https://gowalker.org/github.com/gin-gonic/gin#Default)：返回Gin的`type Engine struct{...}`，里面包含`RouterGroup`，相当于创建一个路由`Handlers`，可以后期绑定各类的路由规则和函数、中间件等
- [router.GET(...){...}](https://gowalker.org/github.com/gin-gonic/gin#IRoutes)：创建不同的HTTP方法绑定到`Handlers`中，也支持POST、PUT、DELETE、PATCH、OPTIONS、HEAD 等常用的Restful方法
- [gin.H{...}](https://gowalker.org/github.com/gin-gonic/gin#H)：就是一个`map[string]interface{}`
- [gin.Context](https://gowalker.org/github.com/gin-gonic/gin#Context)：`Context`是`gin`中的上下文，它允许我们在中间件之间传递变量、管理流、验证JSON请求、响应JSON请求等，在`gin`中包含大量`Context`的方法，例如我们常用的`DefaultQuery`、`Query`、`DefaultPostForm`、`PostForm`等等

#####  &http.Server 和 ListenAndServe？

1、http.Server：

```
type Server struct {
    Addr    string
    Handler Handler
    TLSConfig *tls.Config
    ReadTimeout time.Duration
    ReadHeaderTimeout time.Duration
    WriteTimeout time.Duration
    IdleTimeout time.Duration
    MaxHeaderBytes int
    ConnState func(net.Conn, ConnState)
    ErrorLog *log.Logger
}
```
- Addr：监听的TCP地址，格式为`:8000`
- Handler：http句柄，实质为`ServeHTTP`，用于处理程序响应HTTP请求
- TLSConfig：安全传输层协议（TLS）的配置
- ReadTimeout：允许读取的最大时间
- ReadHeaderTimeout：允许读取请求头的最大时间
- WriteTimeout：允许写入的最大时间
- IdleTimeout：等待的最大时间
- MaxHeaderBytes：请求头的最大字节数
- ConnState：指定一个可选的回调函数，当客户端连接发生变化时调用
- ErrorLog：指定一个可选的日志记录器，用于接收程序的意外行为和底层系统错误；如果未设置或为`nil`则默认以日志包的标准日志记录器完成（也就是在控制台输出）

2、 ListenAndServe：

```
func (srv *Server) ListenAndServe() error {
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```
开始监听服务，监听TCP网络地址，Addr和调用应用程序处理连接上的请求。

我们在源码中看到`Addr`是调用我们在`&http.Server`中设置的参数，因此我们在设置时要用`&`，我们要改变参数的值，因为我们`ListenAndServe`和其他一些方法需要用到`&http.Server`中的参数，他们是相互影响的。

3、 `http.ListenAndServe`和 [连载一](https://segmentfault.com/a/1190000013297625#articleHeader5) 的`r.Run()`有区别吗？ 

我们看看`r.Run`的实现：
```
func (engine *Engine) Run(addr ...string) (err error) {
    defer func() { debugPrintError(err) }()

    address := resolveAddress(addr)
    debugPrint("Listening and serving HTTP on %s\n", address)
    err = http.ListenAndServe(address, engine)
    return
}
```

通过分析源码，得知**本质上没有区别**，同时也得知了启动`gin`时的监听debug信息在这里输出。

4、 为什么Demo里会有`WARNING`？

首先我们可以看下`Default()`的实现
```
// Default returns an Engine instance with the Logger and Recovery middleware already attached.
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```

大家可以看到默认情况下，已经附加了日志、恢复中间件的引擎实例。并且在开头调用了`debugPrintWARNINGDefault()`，而它的实现就是输出该行日志

```
func debugPrintWARNINGDefault() {
	debugPrint(`[WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.
`)
}
```

而另外一个`Running in "debug" mode. Switch to "release" mode in production.`，是运行模式原因，并不难理解，已在配置文件的管控下 :-)，运维人员随时就可以修改它的配置。

5、 Demo的`router.GET`等路由规则可以不写在`main`包中吗？

我们发现`router.GET`等路由规则，在Demo中被编写在了`main`包中，感觉很奇怪，我们去抽离这部分逻辑！

在`go-gin-example`下`routers`目录新建`router.go`文件，写入内容：

```
package routers

import (
    "github.com/gin-gonic/gin"
    
    "github.com/EDDYCJY/go-gin-example/pkg/setting"
)

func InitRouter() *gin.Engine {
    r := gin.New()

    r.Use(gin.Logger())

    r.Use(gin.Recovery())

    gin.SetMode(setting.RunMode)

    r.GET("/test", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "test",
        })
    })

    return r
}
```

修改`main.go`的文件内容：
```
package main

import (
	"fmt"
	"net/http"

	"github.com/EDDYCJY/go-gin-example/routers"
	"github.com/EDDYCJY/go-gin-example/pkg/setting"
)

func main() {
	router := routers.InitRouter()

	s := &http.Server{
		Addr:           fmt.Sprintf(":%d", setting.HTTPPort),
		Handler:        router,
		ReadTimeout:    setting.ReadTimeout,
		WriteTimeout:   setting.WriteTimeout,
		MaxHeaderBytes: 1 << 20,
	}

	s.ListenAndServe()
}
```

当前目录结构：
```
go-gin-example/
├── conf
│   └── app.ini
├── main.go
├── middleware
├── models
│   └── models.go
├── pkg
│   ├── e
│   │   ├── code.go
│   │   └── msg.go
│   ├── setting
│   │   └── setting.go
│   └── util
│       └── pagination.go
├── routers
│   └── router.go
├── runtime
```

重启服务，执行 `curl 127.0.0.1:8000/test `查看是否正确返回。

下一节，我们将以我们的 Demo 为起点进行修改，开始编码！



## 参考

### 本系列示例代码
- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)



## 关于

### 修改记录

- 第一版：2018年02月16日发布文章
- 第二版：2019年10月01日修改文章

## ？

如果有任何疑问或错误，欢迎在 [issues](https://github.com/EDDYCJY/blog) 进行提问或给予修正意见，如果喜欢或对你有所帮助，欢迎 Star，对作者是一种鼓励和推进。

### 我的公众号 

![image](https://image.eddycjy.com/8d0b0c3a11e74efd5fdfd7910257e70b.jpg)
