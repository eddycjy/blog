---
title: "Go 错误处理：用 panic 取代 err != nil 的模式"
date: 2020-12-12T17:21:42+08:00
toc: true
images:
tags: 
  - go
---

前段时间我分享了文章 《先睹为快，Go2 Error 的挣扎之路》后，和一位朋友进行了一次深度交流，他给我分享了他们项目组对于 Go 错误处理的方式调整。

简单来讲，就是在业务代码中使用 `panic` 的方式来替代 “永无止境” 的 `if err != nil`。这就是今天本文的重点内容，我们一起来看看是怎么做，又有什么优缺点。

## 为什么想替换

在 Go 语言中 `if err != nil` 写的太多，还要管方法声明各种，嫌麻烦又不方便：

```
err := foo()
if err != nil {
     //do something..
     return err
}

err := foo()
if err != nil {
     //do something..
     return err
}

err := foo()
if err != nil {
     //do something..
     return err
}

err := foo()
if err != nil {
     //do something..
     return err
}
```

上述还是示例代码，比较直面。若是在工程实践，还得各种 package 跳来跳去加 `if err != nil`，总的来讲比较繁琐，要去关心整体的上下游。更具体的就不赘述了，可以翻看我先前的文章。

## 怎么替换 err != nil

不想写 `if err != nil` 的代码，方式之一就是用 `panic` 来替代他。示例代码如下：

```
func GetFish(db *sql.DB, name string) []string {
	rows, err := db.Query("select name from users where `name` = ?", name)
	if err != nil {
		panic(err)
	}
	defer rows.Close()

	var names []string
	for rows.Next() {
		var name string
		err := rows.Scan(&name)
		if err != nil {
			panic(err)
		}

		names = append(names, name)
	}

	err = rows.Err()
	if err != nil {
		panic(err)
	}

	return names
}
``` 

在上述业务代码中，我们通过 `panic` 的方式取代了 `return err` 的函数返回，自然其所关联的下游业务代码也就不需要编写 `if err != nil` 的代码：

```
func main() {
	fish1 := GetFish(db, "煎鱼")
	fish2 := GetFish(db, "咸鱼")
	fish3 := GetFish(db, "摸鱼")
	...
}
```

同时在转换为使用 `panic` 模式的错误机制后，我们必须要在外层增加 `recover` 方法：

```
func AppRecovery() gin.HandlerFunc {
	return func(c *gin.Context) {
		defer func() {
			if err := recover(); err != nil {
				if _, ok := err.(AppErr); ok {
					// do something...
				} else {
					panic(err)
				}
			}
		}()
	}
}
```

每次 `panic` 后根据其抛出的错误进行断言，识别是否定制的 `AppErr` 错误类型，若是则可以进行一系列的处理动作。否则可继续向上 `panic` 抛出给顶级的 `Recovery` 方法进行处理。

这就是一个相对完整的 `panic` 错误链路处理了。

## 优缺点

- 从优点上来讲：

    - 整体代码结构看起来更加的简洁，仅专注于实现逻辑即可。

    - 不需要关注和编写冗杂的 `if err != nil` 的错误处理代码。

- 从缺点上来讲：

    - 认知负担的增加，需要参加项目的每一个新老同学都清楚该模式，要做一个基本规范或培训。

    - 存在一定的性能开销，每次 `panic` 都存在用户态的上下文切换。

    - 存在一定的风险性，一旦 `panic` 没有 `recover` 住，就会导致事故。

    - Go 官方并不推荐，与 `panic` 本身的定义相违背，也就是 `panic` 与 `error` 的概念混淆。

## 总结

在今天这篇文章给大家分享了如何使用 `panic` 的方式来处理 Go 的错误，其必然有利必有有弊，需要做一个权衡了。

你们团队有没有为了 Go 错误处理做过什么新的调整呢？欢迎在留言区交流和分享。