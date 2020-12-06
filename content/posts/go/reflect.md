---
title: "解密 Go 语言反射 reflect"
date: 2020-11-07T15:01:51+08:00
draft: true
toc: true
images:
tags: 
  - go
---

在所有的语言中，反射这一功能基本属于必不可少的模块。但虽说 “反射” 这个词让人根深蒂固，但更多的还是 WHY。反射到底是什么？为什么反射性能不好，甚至有人说他 “差劲”？反射又是怎么实现的？

今天我们通过这篇文章来一一揭晓，以 Go 语言为例，了解反射到底为何物，其底层又是如何实现的。

## 一个例子

从 pkg.go.dev 上获取一个最常见的 reflect 标准库例子，如下：

```
import (
	"fmt"
	"reflect"
)

func main() {
	rv := []interface{}{"hi", 42, func() {}}
	for _, v := range rv {
		switch v := reflect.ValueOf(v); v.Kind() {
		case reflect.String:
			fmt.Println(v.String())
		case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
			fmt.Println(v.Int())
		default:
			fmt.Printf("unhandled kind %s", v.Kind())
		}
	}
}
```

其输出结果：

```
hi
42
unhandled kind func
```

在程序中主要是声明了 rv 变量，变量类型为 `interface{}`，其包含 3 个不同类型的值。其使用方式常见于变量并不知道入参者具体的基本类型是什么，那么就会用 `interface{}` 类型。

这时候又会引出一个新的问题，既然入参是 `interface{}`，那么出参时呢？毕竟 Go 语言是强类型语言，假设你需要根据其参数进行某些操作，那么必然离不开类型的判断，这时候就要用到反射，也就是 reflect 标准库。

## 反射三大定律

### 第一定律 

反射可以从接口值得到反射对象（Reflection goes from interface value to reflection object）。



```
func main() {
	var x float64 = 3.4
	fmt.Println("type:", reflect.TypeOf(x))
}
```

输出

```
type: float64
```

### 第二定律

反射可以从反射对象得到接口值（Reflection goes from reflection object to interface value）

### 第三定律

要修改反射对象，该值必须可以修改（To modify a reflection object, the value must be settable）

## Go reflect

reflect 标准库中，最常见使用次数最频繁的莫过于 `reflect.Type` 和 `reflect.Value` 类型所关联的方法：

- `TypeOf` 方法：用于提取入参值的**类型信息**。

- `ValueOf` 方法：用于提取存储的变量的**值信息**。

### reflect.TypeOf

```
func main() {
	blog := Blog{"煎鱼"}
	typeof := reflect.TypeOf(blog)
	fmt.Println(typeof.String())
}
```

输出结果：

```
main.Blog
```

从输出结果中可得出 `reflect.TypeOf` 成功解析出 `blog` 变量的类型是 `main.Blog`，也就是连 package 都知道了。通过人识别的角度来看似乎很正常，但程序就不是这样了。

他是怎么知道 “他” 是哪个 package 下的什么呢？我们一起追一下源码看看：

```
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
```

从源码阶段来看，`TypeOf` 方法中主要涉及三块操作，分别如下：

1. 使用 `unsafe.Pointer` 方法获取任意类型且可寻址的指针值。

2. 利用 `emptyInterface` 类型进行强制的 `interface` 类型转换。

3. 调用 `toType` 方法转换为可供外部使用的 `Type` 类型。

而这之中信息量最大的是 `emptyInterface` 结构体中的 `rtype` 类型：

```
type rtype struct {
	size       uintptr
	ptrdata    uintptr 
	hash       uint32 
	tflag      tflag 
	align      uint8  
	fieldAlign uint8  
	kind       uint8   
	equal     func(unsafe.Pointer, unsafe.Pointer) bool
	gcdata    *byte  
	str       nameOff 
	ptrToThis typeOff 
}
```

在使用上最重要的是 `rtype` 类型，其实现了 `Type` 类型的所有接口方法，因此他可以直接作为 `Type` 类型返回，而 `Type` 实际上是一个接口实现，其包含了获取一个类型所必要的所有方法：

```
type Type interface {
	// 适用于所有类型
	// 返回该类型内存对齐后所占用的字节数
	Align() int

	// 仅作用于 strcut 类型
	// 返回该类型内存对齐后所占用的字节数
	FieldAlign() int

	// 返回该类型的方法集中的第 i 个方法
	Method(int) Method

	// 根据方法名获取对应方法集中的方法
	MethodByName(string) (Method, bool)

	// 返回该类型的方法集中导出的方法的数量。
	NumMethod() int

	// 返回该类型的名称
	Name() string

	// 返回定义的类型的包路径，即模块的导入路径。
	PkgPath() string

	// 返回该类型的大小，与 unsafe.Sizeof 方法类型
	Size() uintptr

	// 返回该类型的字符串表现形式
	String() string

	// 返回该类型的 Kind 类型
	Kind() Kind

	// 判断该类型是否实现了 rtype 所需的接口方法
	Implements(u Type) bool

	// 该类型是否可以赋值给 u（rtype）
	AssignableTo(u Type) bool

	// 该类型是否可以转换为 u（rtype）
	ConvertibleTo(u Type) bool

	// 返回该类型是否可以进行比较
	Comparable() bool

	// 该类型所占用的位数（特定类型支持）
	Bits() int

	// 仅支持 channel 类型
	// 返回 channel 类型的数据方向
	ChanDir() ChanDir

	// 仅支持 func 类型
	// 返回的类型是否是可变参数，例如：`func(x int, y ... float64)`。
	IsVariadic() bool

 	// 仅支持 Array，Chan，Map，Ptr 或 Slice 类型
	// 返回类型的元素类型
	Elem() Type

	// 返回 struct 类型的第 i 个字段
	Field(i int) StructField

	// 返回对应的嵌套的 struct 字段
	FieldByIndex(index []int) StructField

	// 通过字段名获取 struct 字段
	FieldByName(name string) (StructField, bool)

	// 通过名称获取对应的 func 方法
	FieldByNameFunc(match func(string) bool) (StructField, bool)

	// 仅支持函数
	// 返回函数类型的第 i 个输入参数的类型。
	In(i int) Type

	// 仅支持 map 类型
	// 返回 map 类型的 key 类型
	Key() Type

	// 仅支持 array 类型
	// 返回 array 类型的总长度
	Len() int

	// 仅支持 struct 类型
	// 返回 struct 类型的字段总数量
	NumField() int

	// 返回函数类型的入参总数量
	NumIn() int

	// 返回函数类型的出参总数量
	NumOut() int

	// 返回函数类型的第 i 个值的类型
	Out(i int) Type

	// 返回已定义的通用类型（大多数值的通用实现）
	common() *rtype

	// 返回没有方法的未定义类型
	uncommon() *uncommonType
}
```

`Type` 接口的方法是真的多，建议大致过一遍，了解清楚有哪些方法，再针对向看就好。主体思想是给自己大脑建立一个索引，便于后续使用到的使用快速到 `pkg.go.dev` 上查询。


### reflect.ValueOf



