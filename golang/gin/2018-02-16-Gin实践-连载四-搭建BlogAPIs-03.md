## 编写Article的API's、Models

项目地址：https://github.com/EDDYCJY/go-gin-example

## 定义接口
本节编写文章的逻辑，我们定义一下接口吧！

- 获取文章列表：GET("/articles")
- 获取指定文章：POST("/articles/:id")
- 新建文章：POST("/articles")
- 更新指定文章：PUT("/articles/:id")
- 删除指定文章：DELETE("/articles/:id")

## 编写路由逻辑
在`routers`的v1版本下，新建`article.go`文件，写入内容：
```
package v1

import (
	"github.com/gin-gonic/gin"
)

//获取单个文章
func GetArticle(c *gin.Context) {
}

//获取多个文章
func GetArticles(c *gin.Context) {
}

//新增文章
func AddArticle(c *gin.Context) {
}

//修改文章
func EditArticle(c *gin.Context) {
}

//删除文章
func DeleteArticle(c *gin.Context) {
}
```

我们打开`routers`下的`router.go`文件，修改文件内容为：
```
package routers

import (
    "github.com/gin-gonic/gin"
    
    "gin-blog/routers/api/v1"
    "gin-blog/pkg/setting"
)

func InitRouter() *gin.Engine {
    ...
    apiv1 := r.Group("/api/v1")
    {
        ...
        //获取文章列表
        apiv1.GET("/articles", v1.GetArticles)
        //获取指定文章
        apiv1.GET("/articles/:id", v1.GetArticle)
        //新建文章
        apiv1.POST("/articles", v1.AddArticle)
        //更新指定文章
        apiv1.PUT("/articles/:id", v1.EditArticle)
        //删除指定文章
        apiv1.DELETE("/articles/:id", v1.DeleteArticle)
    }

    return r
}
```

当前目录结构：
```
gin-blog/
├── conf
│   └── app.ini
├── main.go
├── middleware
├── models
│   ├── models.go
│   └── tag.go
├── pkg
│   ├── e
│   │   ├── code.go
│   │   └── msg.go
│   ├── setting
│   │   └── setting.go
│   └── util
│       └── pagination.go
├── routers
│   ├── api
│   │   └── v1
│   │       ├── article.go
│   │       └── tag.go
│   └── router.go
├── runtime

```
在基础的路由规则配置结束后，我们**开始编写我们的接口**吧！

---

## 编写models逻辑
创建`models`目录下的`article.go`，写入文件内容：
```
package models

import (
	"github.com/jinzhu/gorm"

	"time"
)

type Article struct {
	Model

	TagID int `json:"tag_id" gorm:"index"`
	Tag   Tag `json:"tag"`

	Title string `json:"title"`
	Desc string `json:"desc"`
	Content string `json:"content"`
	CreatedBy string `json:"created_by"`
	ModifiedBy string `json:"modified_by"`
	State int `json:"state"`
}


func (article *Article) BeforeCreate(scope *gorm.Scope) error {
    scope.SetColumn("CreatedOn", time.Now().Unix())

    return nil
}

func (article *Article) BeforeUpdate(scope *gorm.Scope) error {
	scope.SetColumn("ModifiedOn", time.Now().Unix())

	return nil
}
```

我们创建了一个`Article struct {}`，与`Tag`不同的是，`Article`多了几项

1. `gorm:index`，用于声明这个字段为索引，如果你使用了自动迁移功能则会有所影响，在不使用则无影响
2. `Tag`字段，实际是一个嵌套的`struct`，它利用`TagID`与`Tag`模型相互关联，在执行查询的时候，能够达到`Article`、`Tag`关联查询的功能
3. `time.Now().Unix()` 返回当前的时间戳


接下来，请确保已对上一章节的内容通读且了解，由于逻辑偏差不会太远，我们本节直接编写这五个接口

---

打开`models`目录下的`article.go`，修改文件内容：
```
package models

import (
	"time"

	"github.com/jinzhu/gorm"
)

type Article struct {
	Model

	TagID int `json:"tag_id" gorm:"index"`
	Tag   Tag `json:"tag"`

	Title string `json:"title"`
	Desc string `json:"desc"`
	Content string `json:"content"`
	CreatedBy string `json:"created_by"`
	ModifiedBy string `json:"modified_by"`
	State int `json:"state"`
}


func ExistArticleByID(id int) bool {
	var article Article
	db.Select("id").Where("id = ?", id).First(&article)

	if article.ID > 0 {
		return true
	}

	return false
}

func GetArticleTotal(maps interface {}) (count int){
	db.Model(&Article{}).Where(maps).Count(&count)

	return
}

func GetArticles(pageNum int, pageSize int, maps interface {}) (articles []Article) {
	db.Preload("Tag").Where(maps).Offset(pageNum).Limit(pageSize).Find(&articles)

	return
}

func GetArticle(id int) (article Article) {
	db.Where("id = ?", id).First(&article)
	db.Model(&article).Related(&article.Tag)

	return 
}

func EditArticle(id int, data interface {}) bool {
	db.Model(&Article{}).Where("id = ?", id).Updates(data)

	return true
}

func AddArticle(data map[string]interface {}) bool {
	db.Create(&Article {
		TagID : data["tag_id"].(int),
		Title : data["title"].(string),
		Desc : data["desc"].(string),
		Content : data["content"].(string),
		CreatedBy : data["created_by"].(string),
		State : data["state"].(int),
	})

	return true
}

func DeleteArticle(id int) bool {
	db.Where("id = ?", id).Delete(Article{})

	return true
}

func (article *Article) BeforeCreate(scope *gorm.Scope) error {
    scope.SetColumn("CreatedOn", time.Now().Unix())

    return nil
}

func (article *Article) BeforeUpdate(scope *gorm.Scope) error {
	scope.SetColumn("ModifiedOn", time.Now().Unix())

	return nil
}
```


在这里，我们拿出三点不同来讲

1、 我们的`Article`是如何关联到`Tag`？？？
```
func GetArticle(id int) (article Article) {
	db.Where("id = ?", id).First(&article)
	db.Model(&article).Related(&article.Tag)

	return 
}
```
能够达到关联，首先是`gorm`本身做了大量的约定俗成
- `Article`有一个结构体成员是`TagID`，就是外键。`gorm`会通过类名+ID的方式去找到这两个类之间的关联关系
- `Article`有一个结构体成员是`Tag`，就是我们嵌套在`Article`里的`Tag`结构体，我们可以通过`Related`进行关联查询

2、 `Preload`是什么东西，为什么查询可以得出每一项的关联`Tag`？
```
func GetArticles(pageNum int, pageSize int, maps interface {}) (articles []Article) {
	db.Preload("Tag").Where(maps).Offset(pageNum).Limit(pageSize).Find(&articles)

	return
}
```

`Preload`就是一个预加载器，它会执行两条SQL，分别是`SELECT * FROM blog_articles;`和`SELECT * FROM blog_tag WHERE id IN (1,2,3,4);`，那么在查询出结构后，`gorm`内部处理对应的映射逻辑，将其填充到`Article`的`Tag`中，会特别方便，并且避免了循环查询

那么有没有别的办法呢，大致是两种
- `gorm`的`Join`
- 循环`Related`

综合之下，还是`Preload`更好，如果你有更优的方案，欢迎说一下 :)

3、 `v.(I)` 是什么？

`v`表示一个接口值，`I`表示接口类型。这个实际就是Golang中的类型断言，用于判断一个接口值的实际类型是否为某个类型，或一个非接口值的类型是否实现了某个接口类型

---

打开`routers`目录下v1版本的`article.go`文件，修改文件内容：
```
package v1

import (
	"net/http"
	"log"

	"github.com/gin-gonic/gin"
	"github.com/astaxie/beego/validation"
	"github.com/Unknwon/com"

	"gin-blog/models"
	"gin-blog/pkg/e"
	"gin-blog/pkg/setting"
	"gin-blog/pkg/util"
)

//获取单个文章
func GetArticle(c *gin.Context) {
    id := com.StrTo(c.Param("id")).MustInt()

    valid := validation.Validation{}
    valid.Min(id, 1, "id").Message("ID必须大于0")

    code := e.INVALID_PARAMS
    var data interface {}
    if ! valid.HasErrors() {
        if models.ExistArticleByID(id) {
            data = models.GetArticle(id)
            code = e.SUCCESS
        } else {
            code = e.ERROR_NOT_EXIST_ARTICLE
        }
    } else {
        for _, err := range valid.Errors {
            logging.Info(err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : data,
    })
}

//获取多个文章
func GetArticles(c *gin.Context) {
    data := make(map[string]interface{})
    maps := make(map[string]interface{})
    valid := validation.Validation{}

    var state int = -1
    if arg := c.Query("state"); arg != "" {
        state = com.StrTo(arg).MustInt()
        maps["state"] = state

        valid.Range(state, 0, 1, "state").Message("状态只允许0或1")
    }

    var tagId int = -1
    if arg := c.Query("tag_id"); arg != "" {
        tagId = com.StrTo(arg).MustInt()
        maps["tag_id"] = tagId

        valid.Min(tagId, 1, "tag_id").Message("标签ID必须大于0")
    } 

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        code = e.SUCCESS

        data["lists"] = models.GetArticles(util.GetPage(c), setting.PageSize, maps)
        data["total"] = models.GetArticleTotal(maps)

    } else {
        for _, err := range valid.Errors {
            logging.Info(err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : data,
    })
}

//新增文章
func AddArticle(c *gin.Context) {
    tagId := com.StrTo(c.Query("tag_id")).MustInt()
    title := c.Query("title")
    desc := c.Query("desc")
    content := c.Query("content")
    createdBy := c.Query("created_by")
    state := com.StrTo(c.DefaultQuery("state", "0")).MustInt()

    valid := validation.Validation{}
    valid.Min(tagId, 1, "tag_id").Message("标签ID必须大于0")
    valid.Required(title, "title").Message("标题不能为空")
    valid.Required(desc, "desc").Message("简述不能为空")
    valid.Required(content, "content").Message("内容不能为空")
    valid.Required(createdBy, "created_by").Message("创建人不能为空")
    valid.Range(state, 0, 1, "state").Message("状态只允许0或1")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        if models.ExistTagByID(tagId) {
            data := make(map[string]interface {})
            data["tag_id"] = tagId
            data["title"] = title
            data["desc"] = desc
            data["content"] = content
            data["created_by"] = createdBy
            data["state"] = state
            
            models.AddArticle(data)
            code = e.SUCCESS
        } else {
            code = e.ERROR_NOT_EXIST_TAG
        }
    } else {
        for _, err := range valid.Errors {
            logging.Info(err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]interface{}),
    })
}

//修改文章
func EditArticle(c *gin.Context) {
    valid := validation.Validation{}

    id := com.StrTo(c.Param("id")).MustInt()
    tagId := com.StrTo(c.Query("tag_id")).MustInt()
    title := c.Query("title")
    desc := c.Query("desc")
    content := c.Query("content")
    modifiedBy := c.Query("modified_by")

    var state int = -1
    if arg := c.Query("state"); arg != "" {
        state = com.StrTo(arg).MustInt()
        valid.Range(state, 0, 1, "state").Message("状态只允许0或1")
    }

    valid.Min(id, 1, "id").Message("ID必须大于0")
    valid.MaxSize(title, 100, "title").Message("标题最长为100字符")
    valid.MaxSize(desc, 255, "desc").Message("简述最长为255字符")
    valid.MaxSize(content, 65535, "content").Message("内容最长为65535字符")
    valid.Required(modifiedBy, "modified_by").Message("修改人不能为空")
    valid.MaxSize(modifiedBy, 100, "modified_by").Message("修改人最长为100字符")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        if models.ExistArticleByID(id) {
            if models.ExistTagByID(tagId) {
                data := make(map[string]interface {})
                if tagId > 0 {
                    data["tag_id"] = tagId
                }
                if title != "" {
                    data["title"] = title
                }
                if desc != "" {
                    data["desc"] = desc
                }
                if content != "" {
                    data["content"] = content
                }

                data["modified_by"] = modifiedBy

                models.EditArticle(id, data)
                code = e.SUCCESS
            } else {
                code = e.ERROR_NOT_EXIST_TAG
            }
        } else {
            code = e.ERROR_NOT_EXIST_ARTICLE
        }
    } else {
        for _, err := range valid.Errors {
            logging.Info(err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]string),
    })
}

//删除文章
func DeleteArticle(c *gin.Context) {
    id := com.StrTo(c.Param("id")).MustInt()

    valid := validation.Validation{}
    valid.Min(id, 1, "id").Message("ID必须大于0")

    code := e.INVALID_PARAMS
    if ! valid.HasErrors() {
        if models.ExistArticleByID(id) {
            models.DeleteArticle(id)
            code = e.SUCCESS
        } else {
            code = e.ERROR_NOT_EXIST_ARTICLE
        }
    } else {
        for _, err := range valid.Errors {
            logging.Info(err.Key, err.Message)
        }
    }

    c.JSON(http.StatusOK, gin.H{
        "code" : code,
        "msg" : e.GetMsg(code),
        "data" : make(map[string]string),
    })
}
```

当前目录结构：
```
gin-blog/
├── conf
│   └── app.ini
├── main.go
├── middleware
├── models
│   ├── article.go
│   ├── models.go
│   └── tag.go
├── pkg
│   ├── e
│   │   ├── code.go
│   │   └── msg.go
│   ├── setting
│   │   └── setting.go
│   └── util
│       └── pagination.go
├── routers
│   ├── api
│   │   └── v1
│   │       ├── article.go
│   │       └── tag.go
│   └── router.go
├── runtime
```
## 验证功能
我们重启服务，执行`go run main.go`，检查控制台输出结果
```
$ go run main.go 
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /api/v1/tags              --> gin-blog/routers/api/v1.GetTags (3 handlers)
[GIN-debug] POST   /api/v1/tags              --> gin-blog/routers/api/v1.AddTag (3 handlers)
[GIN-debug] PUT    /api/v1/tags/:id          --> gin-blog/routers/api/v1.EditTag (3 handlers)
[GIN-debug] DELETE /api/v1/tags/:id          --> gin-blog/routers/api/v1.DeleteTag (3 handlers)
[GIN-debug] GET    /api/v1/articles          --> gin-blog/routers/api/v1.GetArticles (3 handlers)
[GIN-debug] GET    /api/v1/articles/:id      --> gin-blog/routers/api/v1.GetArticle (3 handlers)
[GIN-debug] POST   /api/v1/articles          --> gin-blog/routers/api/v1.AddArticle (3 handlers)
[GIN-debug] PUT    /api/v1/articles/:id      --> gin-blog/routers/api/v1.EditArticle (3 handlers)
[GIN-debug] DELETE /api/v1/articles/:id      --> gin-blog/routers/api/v1.DeleteArticle (3 handlers)
```

使用`Postman`检验接口是否正常（大家可以选用合适的参数传递方式，此处为了方便展示我选用了URL传参），

- POST：http://127.0.0.1:8000/api/v1/articles?tag_id=1&title=test1&desc=test-desc&content=test-content&created_by=test-created&state=1
- GET：http://127.0.0.1:8000/api/v1/articles
- GET：http://127.0.0.1:8000/api/v1/articles/1
- PUT：http://127.0.0.1:8000/api/v1/articles/1?tag_id=1&title=test-edit1&desc=test-desc-edit&content=test-content-edit&modified_by=test-created-edit&state=0
- DELETE：http://127.0.0.1:8000/api/v1/articles/1


至此，我们的API's编写就到这里，下一节我们将介绍另外的一些技巧！

## 参考
### 本系列示例代码
- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)
