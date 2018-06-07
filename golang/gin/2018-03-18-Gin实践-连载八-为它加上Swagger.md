# 为它加上Swagger

一个好的 `API's`，必然离不开一个好的`API`文档

要开发纯手写 `API` 文档，不存在的 :=)

项目地址：https://github.com/EDDYCJY/go-gin-example

## 安装 swag
1、go get
```
$ go get -u github.com/swaggo/swag/cmd/swag
```
若 `$GOPATH/bin` 没有加入`$PATH`中，你需要执行将其可执行文件移动到`$GOBIN`下
```
mv $GOPATH/bin/swag /usr/local/go/bin
```

2、gopm get

该包有引用`golang.org`上的包，若无科学上网，你可以使用 [gopm](https://gopm.io/) 进行安装
```
gopm get -g -v github.com/swaggo/swag/cmd/swag

cd $GOPATH/src/github.com/swaggo/swag/cmd/swag

go install
```

同理将其可执行文件移动到`$GOBIN`下


### 验证是否安装成功
```
$ swag -v
swag version v1.1.1
```

## 安装 gin-swagger
```
$ go get -u github.com/swaggo/gin-swagger

$ go get -u github.com/swaggo/gin-swagger/swaggerFiles
```

注：三个包都有一定大小，安装需要等一会或要科学上网

## 初始化

## 编写API注释

`Swagger` 中需要将相应的注释或注解编写到方法上，再利用生成器自动生成说明文件


`gin-swagger` 给出的范例：
```
// @Summary Add a new pet to the store
// @Description get string by ID
// @Accept  json
// @Produce  json
// @Param   some_id     path    int     true        "Some ID"
// @Success 200 {string} string	"ok"
// @Failure 400 {object} web.APIError "We need ID!!"
// @Failure 404 {object} web.APIError "Can not find ID"
// @Router /testapi/get-string-by-int/{some_id} [get]
```

我们可以参照 `Swagger` 的注解规范和范例去编写

```
// @Summary 新增文章标签
// @Produce  json
// @Param name query string true "Name"
// @Param state query int false "State"
// @Param created_by query int false "CreatedBy"
// @Success 200 {string} json "{"code":200,"data":{},"msg":"ok"}"
// @Router /api/v1/tags [post]
func AddTag(c *gin.Context) {
```

```
// @Summary 修改文章标签
// @Produce  json
// @Param id param int true "ID"
// @Param name query string true "ID"
// @Param state query int false "State"
// @Param modified_by query string true "ModifiedBy"
// @Success 200 {string} json "{"code":200,"data":{},"msg":"ok"}"
// @Router /api/v1/tags/{id} [put]
func EditTag(c *gin.Context) {
```

参考的注解可见 [gin-blog](https://github.com/EDDYCJY/go-gin-example)

## 生成

我们进入到`gin-blog`的项目根目录中，执行初始化命令
```
[$ gin-blog]# swag init
2018/03/13 23:32:10 Generate swagger docs....
2018/03/13 23:32:10 Generate general API Info
2018/03/13 23:32:10 create docs.go at  docs/docs.go

```

完毕后会在项目根目录下生成`docs`
```
docs/
├── docs.go
└── swagger
    ├── swagger.json
    └── swagger.yaml

```

我们可以检查 `docs.go` 文件中的 `doc` 变量，详细记载中我们文件中所编写的注解和说明
![docs.go](https://sfault-image.b0.upaiyun.com/285/983/2859833321-5aade21608b65_articlex)

## 验证

大功告成，访问一下 `http://127.0.0.1:8000/swagger/index.html`， 查看 `API` 文档生成是否正确

![swagger](https://sfault-image.b0.upaiyun.com/151/604/1516043512-5aade24395bd8_articlex)

## 参考
### 本系列示例代码
- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)
