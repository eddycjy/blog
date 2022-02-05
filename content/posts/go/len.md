---
title: "迷惑了，Go len() 是怎么计算出来的？"
date: 2021-12-31T12:54:58+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

最近看到了一个很有意思的话题，我们平时常常会用 Go 的内置函数 `len` 去获取各种 map、slice 的长度，那他是怎么实现的呢？

正当我想去看看 `len` 的具体实现时，一展身手，却发现竟然是个空方法：

```golang
func len(v Type) int
```

看注解也没有 link 到其他 runtime 函数，那么 len 函数是如何被调用的呢？

先前看国外讨论 Go 计算 len 的文章时做了一些翻译和笔记（底下有参考链接），在此分享给大家，共同进步。

## 谜底

今天就由煎鱼带大家一同解开这个谜底。既然是谜底，那就一开始就揭开。

其实 Go 语言中并没有 len 函数的具体实现代码，他其实**是 Go 编译器的 "魔法" ，不是实际的函数调用**。

接下来将展开这部分，我们可以更深入地了解 Go 编译器的内部工作原理。

## 编译器

在 Go 编译器编译时会解析命令行参数中指定的标志和 Go 源文件，对解析后的 Go 包进行类型检查，将函数编译为机器代码。代码，最后将编译后的包定义写到磁盘上。

内部定义基本类型、内置函数和操作函数的阶段是在 types/universe.go 当中。同时会进行内置函数和具体的操作符匹配，可以明确知道内置函数 len 对应的是 OLEN：

```golang
var builtinFuncs = [...]struct {
	name string
	op   Op
}{
	{"append", OAPPEND},
	{"cap", OCAP},
	{"close", OCLOSE},
	{"complex", OCOMPLEX},
	{"copy", OCOPY},
	{"delete", ODELETE},
	{"imag", OIMAG},
	{"len", OLEN},
	...
}
```

在编译时，上分为五个阶段进行类型检查：
- 第一阶段：常量、类型、以及函数的名称和类型。
- 第二阶段：变量赋值、接口赋值、别名声明。
- 第三阶段：类型检查函数体。
- 第四阶段：检查外部声明。
- 第五阶段：检查类型的地图键，未使用的导入。

如果最后一个类型检查阶段遇到 len 函数，就会转换为 UnaryExpr 类型，一个 UnaryExpr 节点代表一个单数表达式，也最终就是不会成为函数调用：

```golang
func typecheck1(n ir.Node, top int) ir.Node {
	if n, ok := n.(*ir.Name); ok {
		typecheckdef(n)
	}

	switch n.Op() {
	...
	case ir.OCAP, ir.OLEN:
		n := n.(*ir.UnaryExpr)
		return tcLenCap(n)
	}
}
```

在调用 `*ir.UnaryExpr` 转换完毕后，会调用 `tcLenCap`，也就是 typecheck，使用 okforlen 数组来验证参数的合法性或发出相关错误信息：

```golang
func tcLenCap(n *ir.UnaryExpr) ir.Node {
	n.X = Expr(n.X)
	n.X = DefaultLit(n.X, nil)
	n.X = implicitstar(n.X)
	...
	var ok bool
	if n.Op() == ir.OLEN {
		ok = okforlen[t.Kind()]
	} else {
		ok = okforcap[t.Kind()]
	}
  
	...
	n.SetType(types.Types[types.TINT])
	return n
}
```

经历过上面的步骤后在对所有内容进行类型检查后，所有函数都将排队等待编译：

```golang
	base.Timer.Start("be", "compilefuncs")
	fcount := int64(0)
	for i := 0; i < len(typecheck.Target.Decls); i++ {
		if fn, ok := typecheck.Target.Decls[i].(*ir.Func); ok {
			enqueueFunc(fn)
			fcount++
		}
	}
	base.Timer.AddEvent(fcount, "funcs")

	compileFunctions()
```

在经过在 buildssa 和 genssa 之后，再深入几层，就会将 AST 树中的 len 表达式转换为 SSA。接着我们就可以看到 Go 语言中的每种类型的长度是怎么获取的。

这块的处理对应 internal/ssagen/ssa.go 的 expr 方法，如下：

```golang
	case ir.OLEN, ir.OCAP:
		n := n.(*ir.UnaryExpr)
		switch {
		case n.X.Type().IsSlice():
			op := ssa.OpSliceLen
			if n.Op() == ir.OCAP {
				op = ssa.OpSliceCap
			}
			return s.newValue1(op, types.Types[types.TINT], s.expr(n.X))
		case n.X.Type().IsString(): // string; not reachable for OCAP
			return s.newValue1(ssa.OpStringLen, types.Types[types.TINT], s.expr(n.X))
		case n.X.Type().IsMap(), n.X.Type().IsChan():
			return s.referenceTypeBuiltin(n, s.expr(n.X))
		default: // array
			return s.constInt(types.Types[types.TINT], n.X.Type().NumElem())
		}
```

若是数组（array）类型，则会调用 `NumElem` 方法来获取长度值：

```golang
type Array struct {
	Elem  *Type 
	Bound int64 
}

func (t *Type) NumElem() int64 {
	t.wantEtype(TARRAY)
	return t.Extra.(*Array).Bound
}
```

若是字典（map）类型或通道（channel），将会调用 `referenceTypeBuiltin` 方法：

```golang
func (s *state) referenceTypeBuiltin(n *ir.UnaryExpr, x *ssa.Value) *ssa.Value {
	lenType := n.Type()
	nilValue := s.constNil(types.Types[types.TUINTPTR])
	cmp := s.newValue2(ssa.OpEqPtr, types.Types[types.TBOOL], x, nilValue)
	b := s.endBlock()
	b.Kind = ssa.BlockIf
	b.SetControl(cmp)
	b.Likely = ssa.BranchUnlikely

	bThen := s.f.NewBlock(ssa.BlockPlain)
	bElse := s.f.NewBlock(ssa.BlockPlain)
	bAfter := s.f.NewBlock(ssa.BlockPlain)
	...
	switch n.Op() {
	case ir.OLEN:
		s.vars[n] = s.load(lenType, x)
	...
	return s.variable(n, lenType)
}
```

该函数的作用是是获取 map 或chan 的内存地址，并以零偏移量引用其结构布局，就像 `unsafe.Pointer(uintptr(unsafe.Pointer(s))` 一样，返回第一个字面字段的值。

那为什么要获取结构体的第一个字段的值呢，应该是和 map 和 chan 的基础数据结构有关：

```golang
type hmap struct {
	count     int 
  ...
}

type hchan struct {
	qcount   uint    
	...
}
```

是因为 map 和 chan 的基础数据结构的第一个字段就表示长度，自然也就通过计算偏移值来获取了。

其他的数据类型，大家可以继续深入代码，再细看就好了。主要还是枚举多同类的数据类型，接着调用相应的方法。

## 总结

每次我们看到内置函数时，总会下意识的以为是在 runtime 内实现的。看不到 runtime 内的实现方法，又会以为是通过注解 link 的方式来解决的。

但需要注意，其实还有像 len 内置函数这种直接编译器转换的，这也是一种不错的优化方式。

## 参考
- https://tpaschalis.github.io/golang-len
- https://stackoverflow.com/questions/28204831/how-do-os-len-and-make-functions-work