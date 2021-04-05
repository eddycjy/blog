---
title: "Go 群友提问：Goroutine 数量控制在多少合适，会影响 GC 和调度？"
date: 2021-04-05T16:08:18+08:00
images:
tags: 
  - go
  - 面试题
---

大家好，我是煎鱼。

前几天在微信群看到几位大佬在讨论一个问题： ”**字符串 len == 0 和 字符串 == "" ，有啥区别**？“

这是一个比较小的细节点，同时也勾起了我的好奇心，因此今天这篇文章就和大家一起研究一下他们两者有没有区别，谁的性能更好一些？

## 测试方法

在测试的方法中，我们分别声明了 `Test1` 和 `Test2` 方法：

```
func Test1() bool {
	var v string
	if v == "" {
		return true
	}
	return false
}

func Test2() bool {
	var v string
	if len(v) == 0 {
		return true
	}
	return false
}
```

在方法内部仅做了简单的变量类型声明，分别以 字符串 == "" 和 字符串 len == 0 为判断依据。

## 测试用例

编写两个方法的 Benchmark，用于后续的性能测试：

```
func BenchmarkTest1(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Test1()
	}
}

func BenchmarkTest2(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Test2()
	}
}
```

## 结果分析

```
$ go test --bench=. -benchmem
goos: darwin
goarch: amd64
BenchmarkTest1-4   	1000000000	         0.305 ns/op	       0 B/op	       0 allocs/op
BenchmarkTest2-4   	1000000000	         0.305 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	_/Users/eddycjy/go-application/awesomeProject/tests	0.688s
```

从多次测试的结果来看，两者比较：
- 性能几乎没有区别，甚至可以出现一模一样的情况。
- 均不涉及内存申请和操作，均为 0/op。说明变量并不是声明了，就有初始化动作的，这块 Go 编译器有做优化。

结果上居然是一样的。根据曹大的提示，我们可以进一步看一下两者的汇编代码，看看具体区别在哪里：

```
$ go tool compile -S main.go
"".main STEXT nosplit size=1 args=0x0 locals=0x0
	0x0000 00000 (main.go:3)	TEXT	"".main(SB), NOSPLIT|ABIInternal, $0-0
	0x0000 00000 (main.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (main.go:3)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (main.go:5)	RET
	0x0000 c3                                               .
go.cuinfo.packagename. SDWARFINFO dupok size=0
	0x0000 6d 61 69 6e                                      main
""..inittask SNOPTRDATA size=24
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 00 00 00 00 00 00 00 00                          ........
gclocals·33cdeccccebe80329f1fdbee7f5874cb SRODATA dupok size=8
	0x0000 01 00 00 00 00 00 00 00   
```

无论是 `len(v) == 0`，又或是 `v == ""` 的判断，其编译出来的汇编代码都是完全一致的。可以明确 Go 编译器在这块做了明确的优化，大概率是直接比对了。

大家有没有其他的看法和拓展呢，欢迎一起来学习和交流。
