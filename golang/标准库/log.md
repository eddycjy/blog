# Golang 源码剖析：log 标准库

## 日志

### 输出

```
2018/09/28 20:03:08 EDDYCJY Blog...
```

### 构成

[日期]<空格>[时分秒]<空格>[内容]<\n>

## 源码剖析

### Logger

```
type Logger struct {
	mu     sync.Mutex 
	prefix string
	flag   int
	out    io.Writer
	buf    []byte
}
```

1. mu：互斥锁，用于确保原子的写入
2. prefix：每行需写入的日志前缀内容
3. flag：设置日志辅助信息（时间、文件名、行号）的写入。可选如下标识位：

```
const (
	Ldate         = 1 << iota       // value: 1
	Ltime                           // value: 2
	Lmicroseconds                   // value: 4
	Llongfile                       // value: 8
	Lshortfile                      // value: 16
	LUTC                            // value: 32
	LstdFlags     = Ldate | Ltime   // value: 3
)
```

- Ldate：当地时区的格式化日期：2009/01/23
- Ltime：当地时区的格式化时间：01:23:23
- Lmicroseconds：在 Ltime 的基础上，增加微秒的时间数值显示
- Llongfile：完整的文件名和行号：/a/b/c/d.go:23
- Lshortfile：当前文件名和行号：d.go：23，会覆盖 Llongfile 标识
- LUTC：如果设置 Ldate 或 Ltime，且设置 LUTC，则优先使用 UTC 时区而不是本地时区
- LstdFlags：Logger 的默认初始值（Ldate 和 Ltime）

4. out：io.Writer
5. buf：用于存储将要写入的日志内容

### New

```
func New(out io.Writer, prefix string, flag int) *Logger {
	return &Logger{out: out, prefix: prefix, flag: flag}
}

var std = New(os.Stderr, "", LstdFlags)
```

New 方法用于初始化 Logger，接受三个初始参数，可以定制化而在 log 包内默认会初始一个 std，它指向标准输入流。而默认的标准输出、标准错误就是显示器（输出到屏幕上），标准输入就是键盘。辅助的时间信息默认为 `Ldate | Ltime`，也就是 `2009/01/23 01:23:23`

```
// os
var (
	Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
	Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
	Stderr = NewFile(uintptr(syscall.Stderr), "/dev/stderr")
)
```

- Stdin：标准输入
- Stdout：标准输出
- Stderr：标准错误

### Getter

- Flags
- Prefix

### Setter

- SetFlags
- SetPrefix
- SetOutput

### Print*, Fatal*, Panic*

```
func Print(v ...interface{}) {
	std.Output(2, fmt.Sprint(v...))
}

func Printf(format string, v ...interface{}) {
	std.Output(2, fmt.Sprintf(format, v...))
}

func Println(v ...interface{}) {
	std.Output(2, fmt.Sprintln(v...))
}

func Fatal(v ...interface{}) {
	std.Output(2, fmt.Sprint(v...))
	os.Exit(1)
}

func Panic(v ...interface{}) {
	s := fmt.Sprint(v...)
	std.Output(2, s)
	panic(s)
}

...
```

这一部分介绍最常用的日志写入方法，从源码可得知 `Xrintln`、`Xrintf` 函数 **换行**、**可变参数**都是通过 `fmt` 标准库的方法去实现的

`Fatal` 和 `Panic` 是通过 `os.Exit(1)`、`panic(s)` 集成实现的。而具体的组装逻辑是通过 `Output` 方法实现的

#### Logger.Output

```
func (l *Logger) Output(calldepth int, s string) error {
	now := time.Now() // get this early.
	var file string
	var line int
	l.mu.Lock()
	defer l.mu.Unlock()
	if l.flag&(Lshortfile|Llongfile) != 0 {
		// Release lock while getting caller info - it's expensive.
		l.mu.Unlock()
		var ok bool
		_, file, line, ok = runtime.Caller(calldepth)
		if !ok {
			file = "???"
			line = 0
		}
		l.mu.Lock()
	}
	l.buf = l.buf[:0]
	l.formatHeader(&l.buf, now, file, line)
	l.buf = append(l.buf, s...)
	if len(s) == 0 || s[len(s)-1] != '\n' {
		l.buf = append(l.buf, '\n')
	}
	_, err := l.out.Write(l.buf)
	return err
}
```

Output 方法，简单来讲就是将写入的日志事件信息组装并输出，它会根据 flag 标识位的不同来使用 `runtime.Caller` 去获取当前 goroutine 所执行的函数文件、行号等调用信息（log 标准库中默认深度为 2）。另外如果结尾不是换行符 `\n`，将自动补全一个换行

#### Logger.formatHeader

```
func (l *Logger) formatHeader(buf *[]byte, t time.Time, file string, line int) {
	*buf = append(*buf, l.prefix...)
	if l.flag&(Ldate|Ltime|Lmicroseconds) != 0 {
		if l.flag&LUTC != 0 {
			t = t.UTC()
		}
		if l.flag&Ldate != 0 {
			year, month, day := t.Date()
			itoa(buf, year, 4)
			*buf = append(*buf, '/')
			itoa(buf, int(month), 2)
			*buf = append(*buf, '/')
			itoa(buf, day, 2)
			*buf = append(*buf, ' ')
		}
		if l.flag&(Ltime|Lmicroseconds) != 0 {
			hour, min, sec := t.Clock()
			itoa(buf, hour, 2)
			*buf = append(*buf, ':')
			itoa(buf, min, 2)
			*buf = append(*buf, ':')
			itoa(buf, sec, 2)
			if l.flag&Lmicroseconds != 0 {
				*buf = append(*buf, '.')
				itoa(buf, t.Nanosecond()/1e3, 6)
			}
			*buf = append(*buf, ' ')
		}
	}
	if l.flag&(Lshortfile|Llongfile) != 0 {
		if l.flag&Lshortfile != 0 {
			short := file
			for i := len(file) - 1; i > 0; i-- {
				if file[i] == '/' {
					short = file[i+1:]
					break
				}
			}
			file = short
		}
		*buf = append(*buf, file...)
		*buf = append(*buf, ':')
		itoa(buf, line, -1)
		*buf = append(*buf, ": "...)
	}
}
```

该方法主要是用于格式化日志头（前缀），根据入参不同的标识位，添加分隔符和对应的值到日志信息中。执行流程如下：

（1）如果不是空值，则将 prefix 写入 buf

（2）如果设置 `Ldate`、`Ltime`、`Lmicroseconds`，则对应将日期和时间写入 buf

（3）如果设置 `Lshortfile`、`Llongfile`，则对应将文件和行号信息写入 buf

#### Logger.itoa

```
func itoa(buf *[]byte, i int, wid int) {
	// Assemble decimal in reverse order.
	var b [20]byte
	bp := len(b) - 1
	for i >= 10 || wid > 1 {
		wid--
		q := i / 10
		b[bp] = byte('0' + i - q*10)
		bp--
		i = q
	}
	// i < 10
	b[bp] = byte('0' + i)
	*buf = append(*buf, b[bp:]...)
}
```

该方法主要用于将整数转换为定长的十进制 ASCII，同时给出负数宽度避免左侧补 0。另外会以相反的顺序组合十进制 

### 如何定制化 Logger

在标准库内，可通过其开放的 New 方法来实现各种各样的自定义 Logger 组件，但是为什么也可以直接 `log.Print*` 等方法呢？

```
func New(out io.Writer, prefix string, flag int) *Logger
```

其实是在标准库内，如果你刚刚细心的看了前面的小节，不难发现其默认实现了一个 Logger 组件

```
var std = New(os.Stderr, "", LstdFlags)
```

这也是一个小小的精妙之处 ⭕️

## 总结

通过查阅 log 标准库的源码，可得知最简单的一个日志包应该如何编写。另外 log 包是在所有涉及到 Logger 的地方都对 `sync.Mutex` 进行操作（以此解决原子问题），其余逻辑均为组装日志信息和转换数值格式，该包较为经典，可以多读几遍 😄

## 问题

为什么在调用 `runtime.Caller` 前要先解锁，后再加锁呢?
