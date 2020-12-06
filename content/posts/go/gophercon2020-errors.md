---
title: "重磅：Go errors 将不会有任何进一步的改进计划"
date: 2020-11-14T16:48:33+08:00
images:
tags: 
  - go
---

今天在 Gophercon2020 上，**Go 1.13 错误提案的作者事后提及他对目前错误格式化的缺失表示遗憾，而且在未来很长的好几年内都不会有任何进一步改进计划**。

对此他本人给出的原因之一是：对于错误处理这一领域特定的问题，在他的能力范围内实在是无法给出一个令所有人都满意的方案。

尽管如此，在他演讲的最后，还是给出了一些关于错误嵌套的建议，即实现 `fmt.Formatter`，图中给出了一个简单的例子，大家可以参考如下代码：

```
type DetailError struct {
	msg, detail string
	err         error
}

func (e *DetailError) Unwrap() error { return e.err }

func (e *DetailError) Error() string {
	if e.err == nil {
		return e.msg
	}
	return e.msg + ": " + e.err.Error()
}

func (e *DetailError) Format(s fmt.State, c rune) {
	if s.Flag('#') && c == 'v' {
		type nomethod DetailError
		fmt.Fprintf(s, "%#v", (*nomethod)(e))
		return
	}
	if !s.Flag('+') || c != 'v' {
		fmt.Fprintf(s, spec(s, c), e.Error())
		return
	}
	fmt.Fprintln(s, e.msg)
	if e.detail != "" {
		io.WriteString(s, "\t")
		fmt.Fprintln(s, e.detail)
	}
	if e.err != nil {
		if ferr, ok := e.err.(fmt.Formatter); ok {
			ferr.Format(s, c)
		} else {
			fmt.Fprintf(s, spec(s, c), e.err)
			io.WriteString(s, "\n")
		}
	}
}

func spec(s fmt.State, c rune) string {
	buf := []byte{'%'}
	for _, f := range []int{'+', '-', '#', ' ', '0'} {
		if s.Flag(f) {
			buf = append(buf, byte(f))
		}
	}
	if w, ok := s.Width(); ok {
		buf = strconv.AppendInt(buf, int64(w), 10)
	}
	if p, ok := s.Precision(); ok {
		buf = append(buf, '.')
		buf = strconv.AppendInt(buf, int64(p), 10)
	}
	buf = append(buf, byte(c))
	return string(buf)
}
```

此处的内容来源于欧神（@changkun）在知识星球里的线上分享，作为 Go 夜读 SIG 成员的一员，借此也安利下咱们《Go 夜读》的星球，欢迎大家一起来学习和分享：

![image](https://image.eddycjy.com/f474866fbaed634f83fa2e3228cfbec6.jpeg)

## 讨论

Go 语言的错误处理机制一直饱受争议，前段时间在 issues 中还长期争吵过一段时间，因此还是维持了目前 `if err != nil` 的方式，也没有什么大改动。

我们再想想，像其他语言的 `try catch` 是一定好吗？毕竟 `try catch` 的方式也有很多人不看好。抛个砖，**如果是你，你会想怎么设计 Go 语言的错误机制？或者说你觉得怎么的处理才算好**？