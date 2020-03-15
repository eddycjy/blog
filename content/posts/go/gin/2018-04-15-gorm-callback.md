---

title:      "「连载十」定制 GORM Callbacks"
date:       2018-04-15 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
    - gin
---

## 涉及知识点

- GORM

## 本文目标

> GORM itself is powered by Callbacks, so you could fully customize GORM as you want

GORM 本身是由回调驱动的，所以我们可以根据需要完全定制 GORM，以此达到我们的目的，如下：

- 注册一个新的回调
- 删除现有的回调
- 替换现有的回调
- 注册回调的顺序

在 GORM 中包含以上四类 Callbacks，我们结合项目选用 “替换现有的回调” 来解决一个小痛点。

## 问题

在 models 目录下，我们包含 tag.go 和 article.go 两个文件，他们有一个问题，就是 BeforeCreate、BeforeUpdate 重复出现了，那难道 100 个文件，就要写一百次吗？

1、tag.go

![image](https://i.loli.net/2018/04/14/5ad20efdba409.jpg)

2、article.go

![image](https://i.loli.net/2018/04/14/5ad20ebacc4c9.jpg)

显然这是不可能的，如果先前你已经意识到这个问题，那挺 OK，但没有的话，现在开始就要改

### 解决

在这里我们通过 Callbacks 来实现功能，不需要一个个文件去编写

### 实现 Callbacks

打开 models 目录下的 models.go 文件，实现以下两个方法：

1、updateTimeStampForCreateCallback

```go
// updateTimeStampForCreateCallback will set `CreatedOn`, `ModifiedOn` when creating
func updateTimeStampForCreateCallback(scope *gorm.Scope) {
    if !scope.HasError() {
        nowTime := time.Now().Unix()
        if createTimeField, ok := scope.FieldByName("CreatedOn"); ok {
            if createTimeField.IsBlank {
                createTimeField.Set(nowTime)
            }
        }

        if modifyTimeField, ok := scope.FieldByName("ModifiedOn"); ok {
            if modifyTimeField.IsBlank {
                modifyTimeField.Set(nowTime)
            }
        }
    }
}
```

在这段方法中，会完成以下功能

- 检查是否有含有错误（db.Error）
- `scope.FieldByName` 通过 `scope.Fields()` 获取所有字段，判断当前是否包含所需字段

```go
for _, field := range scope.Fields() {
    if field.Name == name || field.DBName == name {
        return field, true
    }
    if field.DBName == dbName {
        mostMatchedField = field
    }
}
```

- `field.IsBlank` 可判断该字段的值是否为空

```go
func isBlank(value reflect.Value) bool {
    switch value.Kind() {
    case reflect.String:
        return value.Len() == 0
    case reflect.Bool:
        return !value.Bool()
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        return value.Int() == 0
    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
        return value.Uint() == 0
    case reflect.Float32, reflect.Float64:
        return value.Float() == 0
    case reflect.Interface, reflect.Ptr:
        return value.IsNil()
    }

    return reflect.DeepEqual(value.Interface(), reflect.Zero(value.Type()).Interface())
}
```

- 若为空则 `field.Set` 用于给该字段设置值，参数为 `interface{}`

2、updateTimeStampForUpdateCallback

```go
// updateTimeStampForUpdateCallback will set `ModifyTime` when updating
func updateTimeStampForUpdateCallback(scope *gorm.Scope) {
    if _, ok := scope.Get("gorm:update_column"); !ok {
        scope.SetColumn("ModifiedOn", time.Now().Unix())
    }
}
```

- `scope.Get(...)` 根据入参获取设置了字面值的参数，例如本文中是 `gorm:update_column` ，它会去查找含这个字面值的字段属性
- `scope.SetColumn(...)` 假设没有指定 `update_column` 的字段，我们默认在更新回调设置 `ModifiedOn` 的值

### 注册 Callbacks

在上面小节我已经把回调方法编写好了，接下来需要将其注册进 GORM 的钩子里，但其本身自带 Create 和 Update 回调，因此调用替换即可

在 models.go 的 init 函数中，增加以下语句

```go
db.Callback().Create().Replace("gorm:update_time_stamp", updateTimeStampForCreateCallback)
db.Callback().Update().Replace("gorm:update_time_stamp", updateTimeStampForUpdateCallback)
```

### 验证

访问 AddTag 接口，成功后检查数据库，可发现 `created_on` 和 `modified_on` 字段都为当前执行时间

访问 EditTag 接口，可发现 `modified_on` 为最后一次执行更新的时间

## 拓展

我们想到，在实际项目中硬删除是较少存在的，那么是否可以通过 Callbacks 来完成这个功能呢？

答案是可以的，我们在先前 `Model struct` 增加 `DeletedOn` 变量

```go
type Model struct {
    ID int `gorm:"primary_key" json:"id"`
    CreatedOn int `json:"created_on"`
    ModifiedOn int `json:"modified_on"`
    DeletedOn int `json:"deleted_on"`
}
```

### 实现 Callbacks

打开 models 目录下的 models.go 文件，实现以下方法：

```go
func deleteCallback(scope *gorm.Scope) {
    if !scope.HasError() {
        var extraOption string
        if str, ok := scope.Get("gorm:delete_option"); ok {
            extraOption = fmt.Sprint(str)
        }

        deletedOnField, hasDeletedOnField := scope.FieldByName("DeletedOn")

        if !scope.Search.Unscoped && hasDeletedOnField {
            scope.Raw(fmt.Sprintf(
                "UPDATE %v SET %v=%v%v%v",
                scope.QuotedTableName(),
                scope.Quote(deletedOnField.DBName),
                scope.AddToVars(time.Now().Unix()),
                addExtraSpaceIfExist(scope.CombinedConditionSql()),
                addExtraSpaceIfExist(extraOption),
            )).Exec()
        } else {
            scope.Raw(fmt.Sprintf(
                "DELETE FROM %v%v%v",
                scope.QuotedTableName(),
                addExtraSpaceIfExist(scope.CombinedConditionSql()),
                addExtraSpaceIfExist(extraOption),
            )).Exec()
        }
    }
}

func addExtraSpaceIfExist(str string) string {
    if str != "" {
        return " " + str
    }
    return ""
}
```

- `scope.Get("gorm:delete_option")` 检查是否手动指定了 delete_option
- `scope.FieldByName("DeletedOn")` 获取我们约定的删除字段，若存在则 `UPDATE` 软删除，若不存在则 `DELETE` 硬删除
- `scope.QuotedTableName()` 返回引用的表名，这个方法 GORM 会根据自身逻辑对表名进行一些处理
- `scope.CombinedConditionSql()` 返回组合好的条件 SQL，看一下方法原型很明了

```go
func (scope *Scope) CombinedConditionSql() string {
    joinSQL := scope.joinsSQL()
    whereSQL := scope.whereSQL()
    if scope.Search.raw {
        whereSQL = strings.TrimSuffix(strings.TrimPrefix(whereSQL, "WHERE ("), ")")
    }
    return joinSQL + whereSQL + scope.groupSQL() +
        scope.havingSQL() + scope.orderSQL() + scope.limitAndOffsetSQL()
}
```

- `scope.AddToVars` 该方法可以添加值作为 SQL 的参数，也可用于防范 SQL 注入

```go
func (scope *Scope) AddToVars(value interface{}) string {
    _, skipBindVar := scope.InstanceGet("skip_bindvar")

    if expr, ok := value.(*expr); ok {
        exp := expr.expr
        for _, arg := range expr.args {
            if skipBindVar {
                scope.AddToVars(arg)
            } else {
                exp = strings.Replace(exp, "?", scope.AddToVars(arg), 1)
            }
        }
        return exp
    }

    scope.SQLVars = append(scope.SQLVars, value)

    if skipBindVar {
        return "?"
    }
    return scope.Dialect().BindVar(len(scope.SQLVars))
}
```

### 注册 Callbacks

在 models.go 的 init 函数中，增加以下删除的回调

```go
db.Callback().Delete().Replace("gorm:delete", deleteCallback)
```

### 验证

重启服务，访问 DeleteTag 接口，成功后即可发现 deleted_on 字段有值

## 小结

在这一章节中，我们结合 GORM 完成了新增、更新、查询的 Callbacks，在实际项目中常常也是这么使用

毕竟，一个钩子的事，就没有必要自己手写过多不必要的代码了

（注意，增加了软删除后，先前的代码需要增加 `deleted_on` 的判断）

## 参考

### 本系列示例代码

- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

### 文档

- [gorm](http://gorm.io/docs/write_plugins.html)

## 关于

### 修改记录

- 第一版：2018 年 02 月 16 日发布文章
- 第二版：2019 年 10 月 01 日修改文章

## ？

如果有任何疑问或错误，欢迎在 [issues](https://github.com/EDDYCJY/blog) 进行提问或给予修正意见，如果喜欢或对你有所帮助，欢迎 Star，对作者是一种鼓励和推进。

### 我的公众号

![image](https://image.eddycjy.com/8d0b0c3a11e74efd5fdfd7910257e70b.jpg)