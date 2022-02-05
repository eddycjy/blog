---
title: "Go 提案：增加泛型版 slices 和 maps 新包"
date: 2021-12-31T12:54:57+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

现在是 2021 年 8 月份了，根据 Go 语言发布周期的 2，8 原则。Go 1.17 即将发布，在写这篇文章时，现在已经进行到了 rc2：

![](https://files.mdnice.com/user/3610/ced4caa3-98ac-4c8a-8077-c72e11f69b56.png)

这意味着离 Go1.18 释出泛型的正式支持又近了一点点，社区中讨论泛型相关的周边功能的声音又多了起来。

今天要讨论的泛型版功能支持也是如此，分别包含：map（#47330）、slice（#45955）、container/set（#47331） 三种通用类型的支持。

我们主要展开 maps 和 slices，其余的都大同小异，理解核心思想就好。

## maps

该提案建议定义一个新的包 maps，它将提供可用于任何类型的 map 的函数：

![](https://files.mdnice.com/user/3610/fc067e0f-0972-4f92-898e-d64404b5bfa4.png)

下面的描述侧重于描述 API 的提供：

```golang
package maps

func Keys[K comparable, V any](m map[K]V) []K

func Values[K comparable, V any](m map[K]V) []V

func Equal[K, V comparable](m1, m2 map[K]V) bool

func EqualFunc[K comparable, V1, V2 any](m1 map[K]V1, m2 map[K]V2, cmp func(V1, V2) bool) bool

func Clear[K comparable, V any](m map[K]V)

func Clone[K comparable, V any](m map[K]V) map[K]V

func Add[K comparable, V any](dst, src map[K]V)

func Filter[K comparable, V any](m map[K]V, keep func(K, V) bool)
```
- Keys：返回 map 的键值。这些键将是一个不确定的顺序。
- Values：返回 map 值。这些值将以不确定的顺序出现。
- Equal：匹配两个 map 是否包含相同的键/值对。
- EqualFunc：EqualFunc 与 Equal 类似，但使用 cmp 进行数值比较。
- Clear：从 map 中清除所有条目，使其为空。
- Clone：返回一个 map 的副本，这是一个浅层克隆。新的键和值使用普通的赋值来设置。
- Add：将 src 中的所有键/值对添加到 dst 中。当 src 中的一个键已经存在于 dst 中时，dst 中的值将被 src 中的键相关的值覆盖。
- Filter：过滤器从 map 中删除任何 keep 返回结果为 false 的键/值对。

## slice

该提案建议定义一个新的包 slices，它将提供可用于任何类型的 slice 的函数：

![](https://files.mdnice.com/user/3610/8086fdea-63ee-4c73-aa4b-95b3c647d8e6.png)

下面的描述侧重于描述 API 的提供：

```golang
package slices

import "constraint"

func Equal[T comparable](s1, s2 []T) bool

func EqualFunc[T1, T2 any](s1 []T1, s2 []T2, eq func(T1, T2) bool) bool

func Compare[T constraints.Ordered](s1, s2 []T) int

func CompareFunc[T any](s1, s2 []T, cmp func(T, T) int) int

func Index[T comparable](s []T, v T) int

func IndexFunc[T any](s []T, f func(T) bool) int

func Contains[T comparable](s []T, v T) bool

func Insert[S constraints.Slice[T], T any](s S, i int, v ...T) S

func Delete[S constraints.Slice[T], T any](s S, i, j int) S

func Clone[S constraints.Slice[T], T any](s S) S
```
- Equal：检查两个切片是否相等，以长度和元素值来比较。
- EqualFunc：检查两个切片是否相等，以所传入的匹配函数来比较。
- Compare/CompareFunc：
    - Compare 比较两个切片 s1 和 s2 的元素。
    - CompareFunc 与 Compare 类似，在每一对元素上使用所传入的比较函数。
- Index：
    - Index 返回值 v 在切片 s 中第一次出现的索引，如果不存在，则返回-1。
    - IndexFunc 返回满足f(c)的第一个元素在s中的索引，如果没有则返回-1。
- Contains：检查值 v 是否存在于切片 s 中。

插入、删除、克隆的 API 比较常见，这里我就不展开了。在通用类型的切片有一些比较特殊的 API：

```golang
func Compact[S constraints.Slice[T], T comparable](s S) S

func CompactFunc[S constraints.Slice[T], T any](s S, cmp func(T, T) bool) S

func Grow[S constraints.Slice[T], T any](s S, n int) S

func Clip[S constraints.Slice[T], T any](s S) S
```
- Compact/CompactFunc：
    - Compact 将连续运行的相等元素替换为单一的副本。这就像 Unix 中的 uniq 命令。会直接修改切片 s 的内容，不会重新创建一个。
    - CompactFunc 与 Compact 方法类似，但是使用一个比较函数来匹配。
- Grow：如果有必要，Grow 会增长切片的容量，以保证另外 n 个元素的空间。在 Grow(n) 之后，至少有 n 个元素可以被添加到分片中而不需要再分配。如果 n 是负数或者太大，无法分配内存，Grow 将陷入恐慌。
- Clip：从切片中移除未使用的容量，返回s[:len(s):len(s)]。

## 总结

如果这些提议被接受，这几个新包将被包含在实现泛型后的第一个Go版本中（我们目前预计将是Go 1.18）。

从 issues 的讨论来看，通用类型的新包支持很大概率会实现，主要争议在实现细节，例如：性能、命名、规范等。

实现后值得期待，又是一次生产力的优化！