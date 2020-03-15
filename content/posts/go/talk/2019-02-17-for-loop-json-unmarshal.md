---

title:      "for-loop 与 json.Unmarshal 性能分析概要"
date:       2019-02-17 12:00:00
author:     "煎鱼"
toc: true
tags:
    - go
---

在项目中，常常会遇到循环交换赋值的数据处理场景，尤其是 RPC，数据交互格式要转为 Protobuf，赋值是无法避免的。一般会有如下几种做法：

- for
- for range
- json.Marshal/Unmarshal

这时候又面临 “选择困难症”，用哪个好？又想代码量少，又担心性能有没有影响啊...

为了弄清楚这个疑惑，接下来将分别编写三种使用场景。来简单看看它们的性能情况，看看谁更 “好”

## 功能代码

```go
...
type Person struct {
	Name   string `json:"name"`
	Age    int    `json:"age"`
	Avatar string `json:"avatar"`
	Type   string `json:"type"`
}

type AgainPerson struct {
	Name   string `json:"name"`
	Age    int    `json:"age"`
	Avatar string `json:"avatar"`
	Type   string `json:"type"`
}

const MAX = 10000

func InitPerson() []Person {
	var persons []Person
	for i := 0; i < MAX; i++ {
		persons = append(persons, Person{
			Name:   "EDDYCJY",
			Age:    i,
			Avatar: "https://github.com/EDDYCJY",
			Type:   "Person",
		})
	}

	return persons
}

func ForStruct(p []Person, count int) {
	for i := 0; i < count; i++ {
		_, _ = i, p[i]
	}
}

func ForRangeStruct(p []Person) {
	for i, v := range p {
		_, _ = i, v
	}
}

func JsonToStruct(data []byte, againPerson []AgainPerson) ([]AgainPerson, error) {
	err := json.Unmarshal(data, &againPerson)
	return againPerson, err
}

func JsonIteratorToStruct(data []byte, againPerson []AgainPerson) ([]AgainPerson, error) {
	var jsonIter = jsoniter.ConfigCompatibleWithStandardLibrary
	err := jsonIter.Unmarshal(data, &againPerson)
	return againPerson, err
}
```

## 测试代码

```go
...
func BenchmarkForStruct(b *testing.B) {
	person := InitPerson()
	count := len(person)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		ForStruct(person, count)
	}
}

func BenchmarkForRangeStruct(b *testing.B) {
	person := InitPerson()

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		ForRangeStruct(person)
	}
}

func BenchmarkJsonToStruct(b *testing.B) {
	var (
		person = InitPerson()
		againPersons []AgainPerson
	)
	data, err := json.Marshal(person)
	if err != nil {
		b.Fatalf("json.Marshal err: %v", err)
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		JsonToStruct(data, againPersons)
	}
}

func BenchmarkJsonIteratorToStruct(b *testing.B) {
	var (
		person = InitPerson()
		againPersons []AgainPerson
	)
	data, err := json.Marshal(person)
	if err != nil {
		b.Fatalf("json.Marshal err: %v", err)
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		JsonIteratorToStruct(data, againPersons)
	}
}
```

## 测试结果

```
BenchmarkForStruct-4              	  500000	      3289 ns/op	       0 B/op	       0 allocs/op
BenchmarkForRangeStruct-4         	  200000	      9178 ns/op	       0 B/op	       0 allocs/op
BenchmarkJsonToStruct-4           	     100	  19173117 ns/op	 2618509 B/op	   40036 allocs/op
BenchmarkJsonIteratorToStruct-4   	     300	   4116491 ns/op	 3694017 B/op	   30047 allocs/op
```

从测试结果来看，性能排名为：for < for range < json-iterator < encoding/json。接下来我们看看是什么原因导致了这样子的排名？

## 性能对比

![image](https://s2.ax1x.com/2020/02/27/3wuywR.png)

### for-loop

在测试结果中，`for range` 在性能上相较 `for` 差。这是为什么呢？在这里我们可以参见 `for range` 的 [实现](https://github.com/gcc-mirror/gcc/blob/master/gcc/go/gofrontend/statements.cc)，伪实现如下：

```go
for_temp := range
len_temp := len(for_temp)
for index_temp = 0; index_temp < len_temp; index_temp++ {
    value_temp = for_temp[index_temp]
    index = index_temp
    value = value_temp
    original body
}
```

通过分析伪实现，可得知 `for range` 相较 `for` 多做了如下事项

#### Expression

```
RangeClause = [ ExpressionList "=" | IdentifierList ":=" ] "range" Expression .
```

在循环开始之前会对范围表达式进行求值，多做了 “解” 表达式的动作，得到了最终的范围值

#### Copy

```go
...
value_temp = for_temp[index_temp]
index = index_temp
value = value_temp
...
```

从伪实现上可以得出，`for range` 始终使用**值拷贝**的方式来生成循环变量。通俗来讲，就是在每次循环时，都会对循环变量重新分配

#### 小结

通过上述的分析，可得知其比 `for` 慢的原因是 `for range` 有额外的性能开销，主要为**值拷贝的动作**导致的性能下降。这是它慢的原因

那么其实在 `for range` 中，我们可以使用 `_` 和 `T[i]` 也能达到和 `for` 差不多的性能。但这可能不是 `for range` 的设计本意了

### json.Marshal/Unmarshal

#### encoding/json

json 互转是在三种方案中最慢的，这是为什么呢？

众所皆知，官方的 `encoding/json` 标准库，是通过大量反射来实现的。那么 “慢”，也是必然的。可参见下述代码：

```go
...
func newTypeEncoder(t reflect.Type, allowAddr bool) encoderFunc {
    ...
	switch t.Kind() {
	case reflect.Bool:
		return boolEncoder
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		return intEncoder
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		return uintEncoder
	case reflect.Float32:
		return float32Encoder
	case reflect.Float64:
		return float64Encoder
	case reflect.String:
		return stringEncoder
	case reflect.Interface:
		return interfaceEncoder
	case reflect.Struct:
		return newStructEncoder(t)
	case reflect.Map:
		return newMapEncoder(t)
	case reflect.Slice:
		return newSliceEncoder(t)
	case reflect.Array:
		return newArrayEncoder(t)
	case reflect.Ptr:
		return newPtrEncoder(t)
	default:
		return unsupportedTypeEncoder
	}
}
```

既然官方的标准库存在一定的 “问题”，那么有没有其他解决方法呢？目前在社区里，大多为两类方案。如下：

- 预编译生成代码（提前确定类型），可以解决运行时的反射带来的性能开销。缺点是增加了预生成的步骤
- 优化序列化的逻辑，性能达到最大化

接下来的实验，我们用第二种方案的库来测试，看看有没有改变。另外也推荐大家了解如下项目：

- [json-iterator/go](https://github.com/json-iterator/go)
- [mailru/easyjson](https://github.com/mailru/easyjson)
- [pquerna/ffjson](https://github.com/pquerna/ffjson)

#### json-iterator/go

目前社区较常用的是 json-iterator/go，我们在测试代码中用到了它

它的用法与标准库 100% 兼容，并且性能有较大提升。我们一起粗略的看下是怎么做到的，如下：

##### reflect2

利用 [modern-go/reflect2](https://github.com/modern-go/reflect2) 减少运行时调度开销

```go
...
type StructDescriptor struct {
	Type   reflect2.Type
	Fields []*Binding
}

...
type Binding struct {
	levels    []int
	Field     reflect2.StructField
	FromNames []string
	ToNames   []string
	Encoder   ValEncoder
	Decoder   ValDecoder
}

type Extension interface {
	UpdateStructDescriptor(structDescriptor *StructDescriptor)
	CreateMapKeyDecoder(typ reflect2.Type) ValDecoder
	CreateMapKeyEncoder(typ reflect2.Type) ValEncoder
	CreateDecoder(typ reflect2.Type) ValDecoder
	CreateEncoder(typ reflect2.Type) ValEncoder
	DecorateDecoder(typ reflect2.Type, decoder ValDecoder) ValDecoder
	DecorateEncoder(typ reflect2.Type, encoder ValEncoder) ValEncoder
}
```

##### struct Encoder/Decoder Cache

类型为 struct 时，只需要反射一次 Name 和 Type，会缓存 struct Encoder 和 Decoder

```go
var typeDecoders = map[string]ValDecoder{}
var fieldDecoders = map[string]ValDecoder{}
var typeEncoders = map[string]ValEncoder{}
var fieldEncoders = map[string]ValEncoder{}
var extensions = []Extension{}

....

fieldNames := calcFieldNames(field.Name(), tagParts[0], tag)
fieldCacheKey := fmt.Sprintf("%s/%s", typ.String(), field.Name())
decoder := fieldDecoders[fieldCacheKey]
if decoder == nil {
	decoder = decoderOfType(ctx.append(field.Name()), field.Type())
}
encoder := fieldEncoders[fieldCacheKey]
if encoder == nil {
	encoder = encoderOfType(ctx.append(field.Name()), field.Type())
}
```

##### 文本解析优化

#### 小结

相较于官方标准库，第三方库 `json-iterator/go` 在运行时上做的更好。这是它快的原因

有个需要注意的点，在 Go1.10 后 `map` 类型与标准库的已经没有太大的性能差异。但是，例如 `struct` 类型等仍然有较大的性能提高

## 总结

在本文中，我们首先进行了性能测试，再分析了不同方案，得知为什么了快慢的原因。那么最终在选择方案时，可以根据不同的应用场景去抉择：

- 对性能开销有较高要求：选用 `for`，开销最小
- 中规中矩：选用 `for range`，大对象慎用
- 量小、占用小、数量可控：选用 `json.Marshal/Unmarshal` 的方案也可以。其**重复代码**少，但开销最大

在绝大多数场景中，使用哪种并没有太大的影响。但作为工程师你应当清楚其利弊。以上就是不同的方案**分析概要**，希望对你有所帮助 :)
