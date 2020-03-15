---

title:      "「连载十二」优化配置结构及实现图片上传"
date:       2018-05-27 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - gin
---

## 知识点

- 重构、调整结构

## 本文目标

这个应用程序跑了那么久了，越来越大，越来越壮，仿佛我们的产品一样，现在它需要进行小范围重构了，以便于后续的使用，这非常重要。

## 前言

一天，产品经理突然跟你说文章列表，没有封面图，不够美观，！）&￥*！&）#&￥*！加一个吧，几分钟的事

你打开你的程序，分析了一波写了个清单：

- 优化配置结构（因为配置项越来越多）
- 抽离 原 logging 的 File 便于公用（logging、upload 各保有一份并不合适）
- 实现上传图片接口（需限制文件格式、大小）
- 修改文章接口（需支持封面地址参数）
- 增加 blog_article （文章）的数据库字段
- 实现 http.FileServer

嗯，你发现要较优的话，需要调整部分的应用程序结构，因为功能越来越多，原本的设计也要跟上节奏

也就是在适当的时候，及时优化

## 优化配置结构

### 一、讲解

在先前章节中，采用了直接读取 KEY 的方式去存储配置项，而本次需求中，需要增加图片的配置项，总体就有些冗余了

我们采用以下解决方法：

- 映射结构体：使用 MapTo 来设置配置参数
- 配置统管：所有的配置项统管到 setting 中

#### 映射结构体（示例）

在 go-ini 中可以采用 MapTo 的方式来映射结构体，例如：

```go
type Server struct {
	RunMode string
	HttpPort int
	ReadTimeout time.Duration
	WriteTimeout time.Duration
}

var ServerSetting = &Server{}

func main() {
    Cfg, err := ini.Load("conf/app.ini")
	if err != nil {
		log.Fatalf("Fail to parse 'conf/app.ini': %v", err)
	}

	err = Cfg.Section("server").MapTo(ServerSetting)
	if err != nil {
		log.Fatalf("Cfg.MapTo ServerSetting err: %v", err)
	}
}
```

在这段代码中，可以注意 ServerSetting 取了地址，为什么 MapTo 必须地址入参呢？

```go
// MapTo maps section to given struct.
func (s *Section) MapTo(v interface{}) error {
	typ := reflect.TypeOf(v)
	val := reflect.ValueOf(v)
	if typ.Kind() == reflect.Ptr {
		typ = typ.Elem()
		val = val.Elem()
	} else {
		return errors.New("cannot map to non-pointer struct")
	}

	return s.mapTo(val, false)
}
```

在 MapTo 中 `typ.Kind() == reflect.Ptr` 约束了必须使用指针，否则会返回 `cannot map to non-pointer struct` 的错误。这个是表面原因

更往内探究，可以认为是 `field.Set` 的原因，当执行 `val := reflect.ValueOf(v)` ，函数通过传递 `v` 拷贝创建了 `val`，但是 `val` 的改变并不能更改原始的 `v`，要想 `val` 的更改能作用到 `v`，则必须传递 `v` 的地址

显然 go-ini 里也是包含修改原始值这一项功能的，你觉得是什么原因呢？

#### 配置统管

在先前的版本中，models 和 file 的配置是在自己的文件中解析的，而其他在 setting.go 中，因此我们需要将其在 setting 中统一接管

你可能会想，直接把两者的配置项复制粘贴到 setting.go 的 init 中，一下子就完事了，搞那么麻烦？

但你在想想，先前的代码中存在多个 init 函数，执行顺序存在问题，无法达到我们的要求，你可以试试

（此处是一个基础知识点）

在 Go 中，当存在多个 init 函数时，执行顺序为：

- 相同包下的 init 函数：按照源文件编译顺序决定执行顺序（默认按文件名排序）
- 不同包下的 init 函数：按照包导入的依赖关系决定先后顺序

所以要避免多 init 的情况，**尽量由程序把控初始化的先后顺序**

### 二、落实

#### 修改配置文件

打开 conf/app.ini 将配置文件修改为大驼峰命名，另外我们增加了 5 个配置项用于上传图片的功能，4 个文件日志方面的配置项

```ini
[app]
PageSize = 10
JwtSecret = 233

RuntimeRootPath = runtime/

ImagePrefixUrl = http://127.0.0.1:8000
ImageSavePath = upload/images/
# MB
ImageMaxSize = 5
ImageAllowExts = .jpg,.jpeg,.png

LogSavePath = logs/
LogSaveName = log
LogFileExt = log
TimeFormat = 20060102

[server]
#debug or release
RunMode = debug
HttpPort = 8000
ReadTimeout = 60
WriteTimeout = 60

[database]
Type = mysql
User = root
Password = rootroot
Host = 127.0.0.1:3306
Name = blog
TablePrefix = blog_
```

#### 优化配置读取及设置初始化顺序

##### 第一步

将散落在其他文件里的配置都删掉，**统一在 setting 中处理**以及**修改 init 函数为 Setup 方法**

打开 pkg/setting/setting.go 文件，修改如下：

```go
package setting

import (
	"log"
	"time"

	"github.com/go-ini/ini"
)

type App struct {
	JwtSecret string
	PageSize int
	RuntimeRootPath string

	ImagePrefixUrl string
	ImageSavePath string
	ImageMaxSize int
	ImageAllowExts []string

	LogSavePath string
	LogSaveName string
	LogFileExt string
	TimeFormat string
}

var AppSetting = &App{}

type Server struct {
	RunMode string
	HttpPort int
	ReadTimeout time.Duration
	WriteTimeout time.Duration
}

var ServerSetting = &Server{}

type Database struct {
	Type string
	User string
	Password string
	Host string
	Name string
	TablePrefix string
}

var DatabaseSetting = &Database{}

func Setup() {
	Cfg, err := ini.Load("conf/app.ini")
	if err != nil {
		log.Fatalf("Fail to parse 'conf/app.ini': %v", err)
	}

	err = Cfg.Section("app").MapTo(AppSetting)
	if err != nil {
		log.Fatalf("Cfg.MapTo AppSetting err: %v", err)
	}

	AppSetting.ImageMaxSize = AppSetting.ImageMaxSize * 1024 * 1024

	err = Cfg.Section("server").MapTo(ServerSetting)
	if err != nil {
		log.Fatalf("Cfg.MapTo ServerSetting err: %v", err)
	}

	ServerSetting.ReadTimeout = ServerSetting.ReadTimeout * time.Second
	ServerSetting.WriteTimeout = ServerSetting.WriteTimeout * time.Second

	err = Cfg.Section("database").MapTo(DatabaseSetting)
	if err != nil {
		log.Fatalf("Cfg.MapTo DatabaseSetting err: %v", err)
	}
}
```

在这里，我们做了如下几件事：

- 编写与配置项保持一致的结构体（App、Server、Database）
- 使用 MapTo 将配置项映射到结构体上
- 对一些需特殊设置的配置项进行再赋值

**需要你去做的事：**

- 将 [models.go](https://github.com/EDDYCJY/go-gin-example/blob/a338ddec103c9506b4c7ed16d9f5386040d99b4b/models/models.go#L23)、[setting.go](https://github.com/EDDYCJY/go-gin-example/blob/a338ddec103c9506b4c7ed16d9f5386040d99b4b/pkg/setting/setting.go#L23)、[pkg/logging/log.go](https://github.com/EDDYCJY/go-gin-example/blob/a338ddec103c9506b4c7ed16d9f5386040d99b4b/pkg/logging/log.go#L32-L37) 的 init 函数修改为 Setup 方法
- 将 [models/models.go](https://github.com/EDDYCJY/go-gin-example/blob/a338ddec103c9506b4c7ed16d9f5386040d99b4b/models/models.go#L23-L39) 独立读取的 DB 配置项删除，改为统一读取 setting
- 将 [pkg/logging/file](https://github.com/EDDYCJY/go-gin-example/blob/a338ddec103c9506b4c7ed16d9f5386040d99b4b/pkg/logging/file.go#L10-L15) 独立的 LOG 配置项删除，改为统一读取 setting

这几项比较基础，并没有贴出来，我希望你可以自己动手，有问题的话可右拐 [项目地址](https://github.com/EDDYCJY/go-gin-example)

##### 第二步

在这一步我们要设置初始化的流程，打开 main.go 文件，修改内容：

```go
func main() {
	setting.Setup()
	models.Setup()
	logging.Setup()

	endless.DefaultReadTimeOut = setting.ServerSetting.ReadTimeout
	endless.DefaultWriteTimeOut = setting.ServerSetting.WriteTimeout
	endless.DefaultMaxHeaderBytes = 1 << 20
	endPoint := fmt.Sprintf(":%d", setting.ServerSetting.HttpPort)

	server := endless.NewServer(endPoint, routers.InitRouter())
	server.BeforeBegin = func(add string) {
		log.Printf("Actual pid is %d", syscall.Getpid())
	}

	err := server.ListenAndServe()
	if err != nil {
		log.Printf("Server err: %v", err)
	}
}
```

修改完毕后，就成功将多模块的初始化函数放到启动流程中了（先后顺序也可以控制）

##### 验证

在这里为止，针对本需求的配置优化就完毕了，你需要执行 `go run main.go` 验证一下你的功能是否正常哦

顺带留个基础问题，大家可以思考下

```go
ServerSetting.ReadTimeout = ServerSetting.ReadTimeout * time.Second
ServerSetting.WriteTimeout = ServerSetting.ReadTimeout * time.Second
```

若将 setting.go 文件中的这两行删除，会出现什么问题，为什么呢？

## 抽离 File

在先前版本中，在 [logging/file.go](https://github.com/EDDYCJY/go-gin-example/blob/a338ddec103c9506b4c7ed16d9f5386040d99b4b/pkg/logging/file.go) 中使用到了 os 的一些方法，我们通过前期规划发现，这部分在上传图片功能中可以复用

### 第一步

在 pkg 目录下新建 file/file.go ，写入文件内容如下：

```go
package file

import (
	"os"
	"path"
	"mime/multipart"
	"io/ioutil"
)

func GetSize(f multipart.File) (int, error) {
	content, err := ioutil.ReadAll(f)

	return len(content), err
}

func GetExt(fileName string) string {
	return path.Ext(fileName)
}

func CheckNotExist(src string) bool {
	_, err := os.Stat(src)

	return os.IsNotExist(err)
}

func CheckPermission(src string) bool {
	_, err := os.Stat(src)

	return os.IsPermission(err)
}

func IsNotExistMkDir(src string) error {
	if notExist := CheckNotExist(src); notExist == true {
		if err := MkDir(src); err != nil {
			return err
		}
	}

	return nil
}

func MkDir(src string) error {
	err := os.MkdirAll(src, os.ModePerm)
	if err != nil {
		return err
	}

	return nil
}

func Open(name string, flag int, perm os.FileMode) (*os.File, error) {
	f, err := os.OpenFile(name, flag, perm)
	if err != nil {
		return nil, err
	}

	return f, nil
}
```

在这里我们一共封装了 7 个 方法

- GetSize：获取文件大小
- GetExt：获取文件后缀
- CheckNotExist：检查文件是否存在
- CheckPermission：检查文件权限
- IsNotExistMkDir：如果不存在则新建文件夹
- MkDir：新建文件夹
- Open：打开文件

在这里我们用到了 `mime/multipart` 包，它主要实现了 MIME 的 multipart 解析，主要适用于 [HTTP](https://tools.ietf.org/html/rfc2388) 和常见浏览器生成的 multipart 主体

multipart 又是什么，[rfc2388](https://tools.ietf.org/html/rfc2388) 的 multipart/form-data 了解一下

### 第二步

我们在第一步已经将 file 重新封装了一层，在这一步我们将原先 logging 包的方法都修改掉

1、打开 pkg/logging/file.go 文件，修改文件内容：

```go
package logging

import (
	"fmt"
	"os"
	"time"

	"github.com/EDDYCJY/go-gin-example/pkg/setting"
	"github.com/EDDYCJY/go-gin-example/pkg/file"
)

func getLogFilePath() string {
	return fmt.Sprintf("%s%s", setting.AppSetting.RuntimeRootPath, setting.AppSetting.LogSavePath)
}

func getLogFileName() string {
	return fmt.Sprintf("%s%s.%s",
		setting.AppSetting.LogSaveName,
		time.Now().Format(setting.AppSetting.TimeFormat),
		setting.AppSetting.LogFileExt,
	)
}

func openLogFile(fileName, filePath string) (*os.File, error) {
	dir, err := os.Getwd()
	if err != nil {
		return nil, fmt.Errorf("os.Getwd err: %v", err)
	}

	src := dir + "/" + filePath
	perm := file.CheckPermission(src)
	if perm == true {
		return nil, fmt.Errorf("file.CheckPermission Permission denied src: %s", src)
	}

	err = file.IsNotExistMkDir(src)
	if err != nil {
		return nil, fmt.Errorf("file.IsNotExistMkDir src: %s, err: %v", src, err)
	}

	f, err := file.Open(src + fileName, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		return nil, fmt.Errorf("Fail to OpenFile :%v", err)
	}

	return f, nil
}
```

我们将引用都改为了 file/file.go 包里的方法

2、打开 pkg/logging/log.go 文件，修改文件内容:

```go
package logging

...

func Setup() {
	var err error
	filePath := getLogFilePath()
	fileName := getLogFileName()
	F, err = openLogFile(fileName, filePath)
	if err != nil {
		log.Fatalln(err)
	}

	logger = log.New(F, DefaultPrefix, log.LstdFlags)
}

...
```

由于原方法形参改变了，因此 openLogFile 也需要调整

## 实现上传图片接口

这一小节，我们开始实现上次图片相关的一些方法和功能

首先需要在 blog_article 中增加字段 `cover_image_url`，格式为 `varchar(255) DEFAULT '' COMMENT '封面图片地址'`

### 第零步

一般不会直接将上传的图片名暴露出来，因此我们对图片名进行 MD5 来达到这个效果

在 util 目录下新建 md5.go，写入文件内容：

```go
package util

import (
	"crypto/md5"
	"encoding/hex"
)

func EncodeMD5(value string) string {
	m := md5.New()
	m.Write([]byte(value))

	return hex.EncodeToString(m.Sum(nil))
}

```

### 第一步

在先前我们已经把底层方法给封装好了，实质这一步为封装 image 的处理逻辑

在 pkg 目录下新建 upload/image.go 文件，写入文件内容：

```go
package upload

import (
	"os"
	"path"
	"log"
	"fmt"
	"strings"
	"mime/multipart"

	"github.com/EDDYCJY/go-gin-example/pkg/file"
	"github.com/EDDYCJY/go-gin-example/pkg/setting"
	"github.com/EDDYCJY/go-gin-example/pkg/logging"
	"github.com/EDDYCJY/go-gin-example/pkg/util"
)

func GetImageFullUrl(name string) string {
	return setting.AppSetting.ImagePrefixUrl + "/" + GetImagePath() + name
}

func GetImageName(name string) string {
	ext := path.Ext(name)
	fileName := strings.TrimSuffix(name, ext)
	fileName = util.EncodeMD5(fileName)

	return fileName + ext
}

func GetImagePath() string {
	return setting.AppSetting.ImageSavePath
}

func GetImageFullPath() string {
	return setting.AppSetting.RuntimeRootPath + GetImagePath()
}

func CheckImageExt(fileName string) bool {
	ext := file.GetExt(fileName)
	for _, allowExt := range setting.AppSetting.ImageAllowExts {
		if strings.ToUpper(allowExt) == strings.ToUpper(ext) {
			return true
		}
	}

	return false
}

func CheckImageSize(f multipart.File) bool {
	size, err := file.GetSize(f)
	if err != nil {
		log.Println(err)
		logging.Warn(err)
		return false
	}

	return size <= setting.AppSetting.ImageMaxSize
}

func CheckImage(src string) error {
	dir, err := os.Getwd()
	if err != nil {
		return fmt.Errorf("os.Getwd err: %v", err)
	}

	err = file.IsNotExistMkDir(dir + "/" + src)
	if err != nil {
		return fmt.Errorf("file.IsNotExistMkDir err: %v", err)
	}

	perm := file.CheckPermission(src)
	if perm == true {
		return fmt.Errorf("file.CheckPermission Permission denied src: %s", src)
	}

	return nil
}
```

在这里我们实现了 7 个方法，如下：

- GetImageFullUrl：获取图片完整访问 URL
- GetImageName：获取图片名称
- GetImagePath：获取图片路径
- GetImageFullPath：获取图片完整路径
- CheckImageExt：检查图片后缀
- CheckImageSize：检查图片大小
- CheckImage：检查图片

这里基本是对底层代码的二次封装，为了更灵活的处理一些图片特有的逻辑，并且方便修改，不直接对外暴露下层

### 第二步

这一步将编写上传图片的业务逻辑，在 routers/api 目录下 新建 upload.go 文件，写入文件内容:

```go
package api

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"github.com/EDDYCJY/go-gin-example/pkg/e"
	"github.com/EDDYCJY/go-gin-example/pkg/logging"
	"github.com/EDDYCJY/go-gin-example/pkg/upload"
)

func UploadImage(c *gin.Context) {
	code := e.SUCCESS
	data := make(map[string]string)

	file, image, err := c.Request.FormFile("image")
	if err != nil {
		logging.Warn(err)
		code = e.ERROR
		c.JSON(http.StatusOK, gin.H{
			"code": code,
			"msg":  e.GetMsg(code),
			"data": data,
		})
	}

	if image == nil {
		code = e.INVALID_PARAMS
	} else {
		imageName := upload.GetImageName(image.Filename)
		fullPath := upload.GetImageFullPath()
		savePath := upload.GetImagePath()

		src := fullPath + imageName
		if ! upload.CheckImageExt(imageName) || ! upload.CheckImageSize(file) {
			code = e.ERROR_UPLOAD_CHECK_IMAGE_FORMAT
		} else {
			err := upload.CheckImage(fullPath)
			if err != nil {
				logging.Warn(err)
				code = e.ERROR_UPLOAD_CHECK_IMAGE_FAIL
			} else if err := c.SaveUploadedFile(image, src); err != nil {
				logging.Warn(err)
				code = e.ERROR_UPLOAD_SAVE_IMAGE_FAIL
			} else {
				data["image_url"] = upload.GetImageFullUrl(imageName)
				data["image_save_url"] = savePath + imageName
			}
		}
	}

	c.JSON(http.StatusOK, gin.H{
		"code": code,
		"msg":  e.GetMsg(code),
		"data": data,
	})
}
```

所涉及的错误码（需在 pkg/e/code.go、msg.go 添加）：

```go
// 保存图片失败
ERROR_UPLOAD_SAVE_IMAGE_FAIL = 30001
// 检查图片失败
ERROR_UPLOAD_CHECK_IMAGE_FAIL = 30002
// 校验图片错误，图片格式或大小有问题
ERROR_UPLOAD_CHECK_IMAGE_FORMAT = 30003
```

在这一大段的业务逻辑中，我们做了如下事情：

- c.Request.FormFile：获取上传的图片（返回提供的表单键的第一个文件）
- CheckImageExt、CheckImageSize 检查图片大小，检查图片后缀
- CheckImage：检查上传图片所需（权限、文件夹）
- SaveUploadedFile：保存图片

总的来说，就是 入参 -> 检查 -》 保存 的应用流程

### 第三步

打开 routers/router.go 文件，增加路由 `r.POST("/upload", api.UploadImage)` ，如：

```go
func InitRouter() *gin.Engine {
	r := gin.New()
    ...
	r.GET("/auth", api.GetAuth)
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
	r.POST("/upload", api.UploadImage)

	apiv1 := r.Group("/api/v1")
	apiv1.Use(jwt.JWT())
	{
		...
	}

	return r
}
```

### 验证

最后我们请求一下上传图片的接口，测试所编写的功能

![image](https://s2.ax1x.com/2020/02/15/1xumb8.jpg)

检查目录下是否含文件（注意权限问题）

```
$ pwd
$GOPATH/src/github.com/EDDYCJY/go-gin-example/runtime/upload/images

$ ll
... 96a3be3cf272e017046d1b2674a52bd3.jpg
... c39fa784216313cf2faa7c98739fc367.jpeg
```

在这里我们一共返回了 2 个参数，一个是完整的访问 URL，另一个为保存路径

## 实现 http.FileServer

在完成了上一小节后，我们还需要让前端能够访问到图片，一般是如下：

- CDN
- http.FileSystem

在公司的话，CDN 或自建分布式文件系统居多，也不需要过多关注。而在实践里的话肯定是本地搭建了，Go 本身对此就有很好的支持，而 Gin 更是再封装了一层，只需要在路由增加一行代码即可

### r.StaticFS

打开 routers/router.go 文件，增加路由 `r.StaticFS("/upload/images", http.Dir(upload.GetImageFullPath()))`，如：

```go
func InitRouter() *gin.Engine {
    ...
	r.StaticFS("/upload/images", http.Dir(upload.GetImageFullPath()))

	r.GET("/auth", api.GetAuth)
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
	r.POST("/upload", api.UploadImage)
    ...
}
```

### 它做了什么

当访问 $HOST/upload/images 时，将会读取到 $GOPATH/src/github.com/EDDYCJY/go-gin-example/runtime/upload/images 下的文件

而这行代码又做了什么事呢，我们来看看方法原型

```go
// StaticFS works just like `Static()` but a custom `http.FileSystem` can be used instead.
// Gin by default user: gin.Dir()
func (group *RouterGroup) StaticFS(relativePath string, fs http.FileSystem) IRoutes {
	if strings.Contains(relativePath, ":") || strings.Contains(relativePath, "*") {
		panic("URL parameters can not be used when serving a static folder")
	}
	handler := group.createStaticHandler(relativePath, fs)
	urlPattern := path.Join(relativePath, "/*filepath")

	// Register GET and HEAD handlers
	group.GET(urlPattern, handler)
	group.HEAD(urlPattern, handler)
	return group.returnObj()
}
```

首先在暴露的 URL 中禁止了 \* 和 : 符号的使用，通过 `createStaticHandler` 创建了静态文件服务，实质最终调用的还是 `fileServer.ServeHTTP` 和一些处理逻辑了

```go
func (group *RouterGroup) createStaticHandler(relativePath string, fs http.FileSystem) HandlerFunc {
	absolutePath := group.calculateAbsolutePath(relativePath)
	fileServer := http.StripPrefix(absolutePath, http.FileServer(fs))
	_, nolisting := fs.(*onlyfilesFS)
	return func(c *Context) {
		if nolisting {
			c.Writer.WriteHeader(404)
		}
		fileServer.ServeHTTP(c.Writer, c.Request)
	}
}
```

#### http.StripPrefix

我们可以留意下 `fileServer := http.StripPrefix(absolutePath, http.FileServer(fs))` 这段语句，在静态文件服务中很常见，它有什么作用呢？

`http.StripPrefix` 主要作用是从请求 URL 的路径中删除给定的前缀，最终返回一个 `Handler`

通常 http.FileServer 要与 http.StripPrefix 相结合使用，否则当你运行：

```go
http.Handle("/upload/images", http.FileServer(http.Dir("upload/images")))
```

会无法正确的访问到文件目录，因为 `/upload/images` 也包含在了 URL 路径中，必须使用：

```go
http.Handle("/upload/images", http.StripPrefix("upload/images", http.FileServer(http.Dir("upload/images"))))
```

#### /\*filepath

到下面可以看到 `urlPattern := path.Join(relativePath, "/*filepath")`，`/*filepath` 你是谁，你在这里有什么用，你是 Gin 的产物吗?

通过语义可得知是路由的处理逻辑，而 Gin 的路由是基于 httprouter 的，通过查阅文档可得到以下信息

```
Pattern: /src/*filepath

 /src/                     match
 /src/somefile.go          match
 /src/subdir/somefile.go   match
```

`*filepath` 将匹配所有文件路径，并且 `*filepath` 必须在 Pattern 的最后

### 验证

重新执行 `go run main.go` ，去访问刚刚在 upload 接口得到的图片地址，检查 http.FileSystem 是否正常

![image](https://s2.ax1x.com/2020/02/15/1xu4Gd.jpg)

## 修改文章接口

接下来，需要你修改 routers/api/v1/article.go 的 AddArticle、EditArticle 两个接口

- 新增、更新文章接口：支持入参 cover_image_url
- 新增、更新文章接口：增加对 cover_image_url 的非空、最长长度校验

这块前面文章讲过，如果有问题可以参考项目的代码 👌

## 总结

在这章节中，我们简单的分析了下需求，对应用做出了一个小规划并实施

完成了清单中的功能点和优化，在实际项目中也是常见的场景，希望你能够细细品尝并针对一些点进行深入学习

## 参考

### 本系列示例代码

- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

## 关于

### 修改记录

- 第一版：2018 年 02 月 16 日发布文章
- 第二版：2019 年 10 月 02 日修改文章

## ？

如果有任何疑问或错误，欢迎在 [issues](https://github.com/EDDYCJY/blog) 进行提问或给予修正意见，如果喜欢或对你有所帮助，欢迎 Star，对作者是一种鼓励和推进。

### 我的公众号

![image](https://image.eddycjy.com/8d0b0c3a11e74efd5fdfd7910257e70b.jpg)