---
title: Go 错误处理指北：pkg／errors 源码解读
author:
  - Go 编程世界
date created: 2025-01-01 14:37:48
category: Golang
tags:
  - "#Golang"
  - "#错误处理"
  - "#编程实践"
  - "#代码优化"
  - "#软件开发"
url:
  - https://mp.weixin.qq.com/s/_BWF5TZP_WMAknI9GNXn-w
description: 本文深入源码，详细解读了Go语言中著名的错误处理包`pkg/errors`的设计与实现。文章从Go错误处理的常见诉求出发，阐述了`pkg/errors`如何帮助开发者附加错误信息和堆栈信息，从而更好地定位和解决问题。通过对`errors.go`、`go113.go`和`stack.go`三个核心文件的源码分析，揭示了`pkg/errors`提供的三种错误类型（`fundamental`、`withStack`、`withMessage`）的原理和用法，以及如何利用`runtime.Callers`和`runtime.FuncForPC`获取调用栈信息。尽管Go 2错误提案遥遥无期，`pkg/errors`仍然是Go错误处理的有力选择，其源码值得深入学习。
status: Finished
---

`pkg/errors` 包在 Go 错误处理生态中可谓大名鼎鼎了，截止目前在 GitHub 上有 8.2k 的 star 量。虽然不是 Go 官方包，但却被很多团队当作事实标准来使用。

本文就来深入到 `pkg/errors` 包源码中，一窥它的设计与实现。

### Go 错误处理诉求

Go 语言的错误处理非常简单，秉承着 Errors are values 的大道至简风格。不过也正是由于简单，也就暴露了 Go 错误处理的些许简陋。

在日常开发中，我们对于错误处理会有两个最普遍的诉求：

1. 附加错误信息：在拿到原有的底层代码或第三方库返回的错误后，我们可能希望附加一些业务信息，比如 `userID`，这样就知道这条错误是由哪个用户产生的。
2. 附加错误堆栈：因为错误堆栈中有出错代码的位置，以及整个调用链路，这会方便我们定位问题。

在错误中附加了这两个信息以后，输出的错误日志就非常有价值了，我们可以根据输出信息快速定位问题。

在 Go 1.13 版本之前，给一个错误附加错误信息可以这样做：

```go
newErr := fmt.Errorf("user %d is err: %s", userID, err)
```

不过，这里存在一个较大的问题，就是新的错误 `newErr` 与原来的 `err` 是**两个完全不同的值**，有时候我们想通过 `newErr` 找到错误的原始根因 `err`，就没办法了。

好在 Go 1.13 版本为 `fmt.Errorf` 提供了 `%w` 动词，能给解决无法通过 `newErr` 找到 `err` 的问题，我们只需要把原来的 `%s` 换成 `%w` 即可：

```go
newErr := fmt.Errorf("user %d is err: %w", userID, err)
```

这样，我们就可以用 `err = errors.Unwrap(newErr)` 拿到错误根因 `err` 了。

但是，Go 依然没有提供获取错误堆栈信息的方法，只能我们自己想办法解决。

幸运的是，以上这两个问题 `pkg/errors` 包都可以帮我们解决。

`pkg/errors` 包使用示例如下：

```go
package main

import (
    "fmt"

    "github.com/pkg/errors"
)

func a() error {
    // 初始错误
    return errors.New("a error")
}

func b() error {
    err := a()
    if err != nil {
        // 包装新的错误并返回
        newErr := errors.WithMessage(err, "b error")
        // newErr := errors.Wrap(err, "b error")

        // 可以从包装后的错误中还原出初始错误
        fmt.Printf("newErr cause == err: %t\n", errors.Cause(newErr) == err)

        return newErr
    }
    return nil
}

func main() {
    err := b()
    if err != nil {
        // %v 打印错误信息
        fmt.Printf("%v\n", err)

        fmt.Println("============================================")

        // %+v 打印错误信息和错误堆栈
        fmt.Printf("%+v\n", err)

        fmt.Println("============================================")

        // 打印错误根因
        fmt.Printf("%v\n", errors.Cause(err))
        return
    }
    fmt.Println("success")
}
```

示例代码中，函数调用链为：`main() -> b() -> a()`。

函数 `a` 直接返回一个错误 `errors.New("a error")`。函数 `b` 在得到错误后，对其进行了包装，附加了新的错误信息 `newErr := errors.WithMessage(err, "b error")`，使用 `errors.Cause(newErr)` 可以从包装后的错误中还原出初始的错误根因。最后在 `main` 函数中，我们分别以不同形式打印了三次错误信息，`%v` 动词可以打印错误信息，`%+v` 可以打印错误信息和错误堆栈。

> NOTE: 示例代码中有一行被注释了 `newErr := errors.Wrap(err, "b error")`，你可以自行尝试下如果使用 `errors.Wrap` 来包装 `err`，最终程序输出结果有何变化。

执行示例代码，输出结果如下：

```sh
$ go run main.go
newErr cause == err: true
b error: a error
============================================
a error
main.a
        /go/blog-go-example/error/pkg-errors/main.go:11
main.b
        /go/blog-go-example/error/pkg-errors/main.go:15
main.main
        /go/blog-go-example/error/pkg-errors/main.go:30
runtime.main
        /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.22.7.darwin-arm64/src/runtime/proc.go:271
runtime.goexit
        /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.22.7.darwin-arm64/src/runtime/asm_arm64.s:1222
b error
============================================
a error
```

输出结果一目了然，使用 `errors.Cause(newErr)` 拿到的错误根因就是 `err`。使用 `errors.WithMessage(err, "b error")` 对 `err` 包装后得到的 `newErr` 使用 `%v` 输出结果为 `b error: a error`，而使用 `%+v` 输出结果则包含了整个错误调用链的堆栈信息。

这就是 `pkg/errors` 的 “威力”，非常好用。

虽然由于 Go 2 错误提案 的存在，现在 `pkg/errors` 包 GitHub 仓库已经归档，但等待 Go 2 演进的路上遥遥无期，`pkg/errors` 包依然是最有竞争力的选择。

所以 `pkg/errors` 仍然推荐使用，并且源码也值得一读。

### pkg/errors 源码解读

`pkg/errors` 提供了这些公开函数供我们使用：

```sh
$ go doc pkg/errors | grep "^func"
functions destructure errors.Wrap into its component operations: annotating an
func As(err error, target interface{}) bool
func Cause(err error) error
func Errorf(format string, args ...interface{}) error
func Is(err, target error) bool
func New(message string) error
func Unwrap(err error) error
func WithMessage(err error, message string) error
func WithMessagef(err error, format string, args ...interface{}) error
func WithStack(err error) error
func Wrap(err error, message string) error
func Wrapf(err error, format string, args ...interface{}) error
```

这些函数的用法，我就不一一演示了，先有个印象，讲解完源码以后，你自己就会用了。

接下来我们一起对源码进行详细解读，看看 `pkg/errors` 底层到底是如何实现的。

`pkg/errors` 项目目录结构如下：

```sh
$ tree github.com/pkg/errors@v0.9.1
errors
├── LICENSE
├── Makefile
├── README.md
├── appveyor.yml
├── bench_test.go
├── errors.go
├── errors_test.go
├── example_test.go
├── format_test.go
├── go113.go
├── go113_test.go
├── json_test.go
├── stack.go
└── stack_test.go

1 directory, 14 files
```

可以发现，此项目采用平铺式目录结构，除了非 Go 代码文件以及测试文件，我们需要关注的 Go 文件就仅有 3 个：`errors.go`、`go113.go` 以及 `stack.go`。

看到这里，你也就可以放宽心了，`pkg/errors` 包源码其实并没有多少。

#### errors.go

我们先来看 `errors.go` 文件。

这个文件中的代码量并不多，即使算上全部的注释，也才不到 `300` 行。

`pkg/errors` 提供的第一个表示错误的结构体 `fundamental` 定义如下：

```go
func New(message string) error {
    return &fundamental{
       msg:   message,
       stack: callers(),
    }
}

func Errorf(format string, args ...interface{}) error {
    return &fundamental{
       msg:   fmt.Sprintf(format, args...),
       stack: callers(),
    }
}

type fundamental struct {
    msg string
    *stack
}

func (f *fundamental) Error() string { return f.msg }

func (f *fundamental) Format(s fmt.State, verb rune) {
    switch verb {
    case 'v':
        if s.Flag('+') {
            io.WriteString(s, f.msg)
            f.stack.Format(s, verb)
            return
        }
        fallthrough
    case 's':
       io.WriteString(s, f.msg)
    case 'q':
        fmt.Fprintf(s, "%q", f.msg)
    }
}
```

`fundamental` 结构体定义非常简单，仅有两个字段：

- `msg` 用来记录错误信息。
- 指针字段 `*stack` 用来记录出错时的错误堆栈信息。这里我们现在仅需要知道 `*stack` 是什么即可，暂且不去深究它的底层实现。

有两个函数都可以构造 `fundamental` 类型的错误：

- `New` 函数通过错误信息 `message` 来构造错误对象，并在内部调用 `callers()` 获取错误堆栈。
- `Errorf` 函数接收变长参数，内部使用 `fmt.Sprintf` 来格式化错误信息。

`New` 其实对标的就是 Go 自带的 `errors.New()`，这两个函数签名相同。而 `Errorf` 对标的是 Go 自带的 `fmt.Errorf()`，两个函数签名也相同。

`pkg/errors` 的函数命名还是比较考究的，与内置库同名。这样能够减少迁移成本，并且降低理解的心智负担。

`fundamental` 错误类还实现了两个方法 `Error` 和 `Format`。

- `Error` 方法没什么好说的，就是用来实现标准的 `error` 接口。这里只返回了 `msg` 信息，没有对错误堆栈信息做任何处理。
- `Format` 方法是格式化输出用的，实现了 `fmt.Formatter` 接口，而参数 `s fmt.State` 也是个接口。

这两个接口定义如下：

```go
type Formatter interface {
    Format(f State, verb rune)
}

type State interface {
    Write(b []byte) (n int, err error)
    Width() (wid int, ok bool)
    Precision() (prec int, ok bool)
    Flag(c int) bool
}
```

当我们使用 `fmt.Printf("%v\n", err)` 格式化打印错误对象信息时，就会调用 `err` 对象的 `Format` 方法。

`Format` 方法的参数 `s fmt.State` 又实现了 `io.Writer` 接口，我们向这个参数中写入的任何内容，都会被替换到 `%v` 占位符处。

而通过 `verb` 参数，我们可以拿到 `fmt.Printf` 的格式化动词，就是 `%` 后面紧挨着的字符，比如 `%s`、`%v` 中的 `s` 和 `v`。

有了 `verb` 就可以定制化不同输出了，`%v` 会被 `Format` 方法内部的第一个 `case` 匹配，如果是 `%+v` 则进入 `if s.Flag('+')` 逻辑。

根据 `Format` 方法源码，可以总结 `fundamental` 类型错误输出格式有如下几种：

| verb | 输出          |
| ---- | ----------- |
| %s   | 错误信息        |
| %v   | 错误信息        |
| %+v  | 错误信息 + 堆栈信息 |
| %q   | 转义后的错误信息    |

> NOTE: 如果你对这几个 `verb` 不是很熟悉，可以查看 Go fmt 文档：https://pkg.go.dev/fmt。

我们再来看下 `pkg/errors` 提供的第二个表示错误的结构体 `withStack`：

```go
func WithStack(err error) error {
    if err == nil {
        return nil
    }
    return &withStack{
        err,
        callers(),
    }
}

type withStack struct {
    error
    *stack
}

func (w *withStack) Cause() error { return w.error }

// Unwrap provides compatibility for Go 1.13 error chains.
func (w *withStack) Unwrap() error { return w.error }

func (w *withStack) Format(s fmt.State, verb rune) {
    switch verb {
    case 'v':
        if s.Flag('+') {
            fmt.Fprintf(s, "%+v", w.Cause())
            w.stack.Format(s, verb)
            return
        }
        fallthrough
    case 's':
        io.WriteString(s, w.Error())
    case 'q':
        fmt.Fprintf(s, "%q", w.Error())
    }
}
```

`withStack` 结构体同样包含两个字段，分别是 `error` 和错误堆栈。顾名思义，这个结构体就是为了给一个错误对象，携带错误堆栈信息用的。

`WithStack` 构造函数也确实是干这件事的，它接收一个 `error`，并返回一个 `withStack` 类型错误，这时已经附加上了错误堆栈信息。

`withStack` 结构体的 `Cause` 方法返回 `error` 字段，表示错误根因，即 `withStack` 错误的初始原因。

至于 `Unwrap` 方法，根据注释可以了解，是为了兼容 Go 1.13 而适配的方法，功能与 `Cause` 一样。

`withStack` 结构体同样定义了 `Format` 方法，这没什么好解释的了。

总结 `withStack` 类型错误输出格式有如下几种：

|verb|输出|
|---|---|
|%s|错误信息|
|%v|错误信息|
|%+v|错误根因 + 堆栈信息|
|%q|转义后的错误信息|

`pkg/errors` 提供的最后一个表示错误的结构体为 `withMessage`：

```go
func WithMessage(err error, message string) error {
    if err == nil {
        return nil
    }
    return &withMessage{
        cause: err,
        msg:   message,
    }
}

func WithMessagef(err error, format string, args ...interface{}) error {
    if err == nil {
        return nil
    }
    return &withMessage{
        cause: err,
        msg:   fmt.Sprintf(format, args...),
    }
}

type withMessage struct {
    cause error
    msg   string
}

func (w *withMessage) Error() string { return w.msg + ": " + w.cause.Error() }
func (w *withMessage) Cause() error  { return w.cause }

// Unwrap provides compatibility for Go 1.13 error chains.
func (w *withMessage) Unwrap() error { return w.cause }

func (w *withMessage) Format(s fmt.State, verb rune) {
    switch verb {
    case 'v':
        if s.Flag('+') {
            fmt.Fprintf(s, "%+v\n", w.Cause())
            io.WriteString(s, w.msg)
            return
        }
        fallthrough
    case 's', 'q':
        io.WriteString(s, w.Error())
    }
}
```

`withMessage` 是给一个错误附加新的错误信息，这里并没有包含错误堆栈信息。

它的方法功能一目了然，无需我多解释什么了。

总结 `withMessage` 类型错误输出格式有如下几种：

|verb|输出|
|---|---|
|%s|错误信息|
|%v|错误信息|
|%+v|错误根因 + 错误信息|
|%q|错误信息|

现在我们将 `pkg/errors` 包提供的三种错误放在一起进行对比：

```go
type fundamental struct {
    msg string
    *stack
}

type withStack struct {
    error
    *stack
}

type withMessage struct {
    cause error
    msg   string
}
```

可以发现，其实这三个错误就是 `error`、`msg` 以及 `stack` 三者的两两组合。

`error` 就是初始错误，`msg` 是附加的错误信息，`stack` 则是附加的堆栈信息。

那么你可能会想，`pkg/errors` 包有没有提供将这三个信息放在一起的方法？

当然有，`errors.go` 源码中剩下的最后三个函数定义如下：

```go
func Wrap(err error, message string) error {
    if err == nil {
        return nil
    }
    err = &withMessage{
        cause: err,
        msg:   message,
    }
    return &withStack{
        err,
        callers(),
    }
}

func Wrapf(err error, format string, args ...interface{}) error {
    if err == nil {
        return nil
    }
    err = &withMessage{
        cause: err,
        msg:   fmt.Sprintf(format, args...),
    }
    return &withStack{
        err,
        callers(),
    }
}

func Cause(err error) error {
    type causer interface {
        Cause() error
    }

    for err != nil {
        cause, ok := err.(causer)
        if !ok {
            break
        }
       err = cause.Cause()
    }
    return err
}
```

`Wrap` 函数，接收一个 `err`，和一个错误信息 `message`，如果 `err` 为 `nil`，直接返回，否则，先使用这两个信息构造 `withMessage` 错误类型，然后再用这个新的 `error` 对象和通过调用 `callers()` 得到的错误堆栈 `*stack` 一起构造一个新的 `withStack` 错误。

`Wrap` 函数最终返回的 `error` 就完整的包含了 `error`、`msg` 以及 `stack` 三个信息。

`Wrapf` 函数同理。

`Cause` 函数则用来递归的拆解嵌套错误，直到取出错误对象的最原始的错误，即错误根因。

以上，就是 `errors.go` 文件的全部代码了。

#### go113.go

然后我们再一起看看 `go113.go` 文件中的代码。

`go113.go` 文件源码如下：

```go
//go:build go1.13
// +build go1.13

package errors

import (
    stderrors "errors"
)

func Is(err, target error) bool { return stderrors.Is(err, target) }

func As(err error, target interface{}) bool { return stderrors.As(err, target) }

func Unwrap(err error) error {
    return stderrors.Unwrap(err)
}
```

根据这个文件的名称，想必你也能猜测出来，这里的代码是与 Go 1.13 版本相关的，构建约束 `// +build go1.13` 也证实了这一点。

`Is`、`As`、`Unwrap` 这三个函数都是 Go 1.13 版本引入的。

- `Is` 函数用来判断两个错误是否为同一个，它用来替代 == 运算符。
- `As` 用来替代错误断言。
- `Unwrap` 用来解包错误，跟 `Cause` 函数作用相同。

这个文件就是用来兼容 Go 1.13 的，并无其他额外功能。

#### stack.go

接下来我们要阅读源码的文件就只剩下 `stack.go` 了。

首先我们要看的就是在 `errors.go` 中见的最多的 `callers()` 函数调用，`callers` 函数就出自这个文件，来看看这个函数是怎么定义的：

```go
func callers() *stack {
    const depth = 32
    var pcs [depth]uintptr
    n := runtime.Callers(3, pcs[:])
    var st stack = pcs[0:n]
    return &st
}
```

`callers` 函数内部使用 `runtime.Callers` 获取当前的调用栈信息。这在代码调试、生成错误报告、记录日志等场景非常有用。

`runtime.Callers` 函数签名如下：

```go
func Callers(skip int, pc []uintptr) int
```

`skip` 参数用于指定要跳过的栈帧数量。`0` 表示包括 `runtime.Callers` 自己的调用帧，`1` 表示从调用 `runtime.Callers` 的函数开始，依此类推。通常设置 `skip` 为 `1` 或更高，以跳过对错误检测或日志记录功能本身的调用。这里设为 `3` 表示跳过 `runtime.Callers`、`callers` 以及 `errors.go` 中定义的 `New`、`Errorf` 等函数调用帧。

`runtime.Callers` 函数会填充传递给它的指针切片 `pc []uintptr` 当前 goroutine 调用栈的程序计数器（PC）值。这个 `pc` 可以用于进一步获取关于每个栈帧的详细信息，例如通过 `runtime.FuncForPC` 函数获取函数名称、行号等，稍后会进行演示讲解。`pc` 切片的大小决定了可以捕获的栈帧的数量。

`runtime.Callers` 函数返回值是 `int` 类型，它记录填充到 `pc` 切片中的项目数。如果实际的调用栈比提供的 `pc` 长度短，返回值就小于 `pc` 切片的长度。

所以，`stack` 类型的变量 `st` 的赋值操作，只取了 `pcs[0:n]`，`n` 是返回的切片长度。

既然 `pcs` 切片变量能赋值给 `stack` 类型变量 `st`，就说明二者类型一致。

`stack` 类型的底层类型也确实是 `[]uintptr`，定义如下：

```go
// stack represents a stack of program counters.
type stack []uintptr
```

`stack` 支持的方法定义如下：

```go
func (s *stack) Format(st fmt.State, verb rune) {
    switch verb {
    case 'v':
        switch {
        case st.Flag('+'):
            for _, pc := range *s {
                f := Frame(pc)
                fmt.Fprintf(st, "\n%+v", f)
            }
        }
    }
}

func (s *stack) StackTrace() StackTrace {
    f := make([]Frame, len(*s))
    for i := 0; i < len(f); i++ {
        f[i] = Frame((*s)[i])
    }
    return f
}
```

你是否记得 `Format` 方法，会在 `error.Format` 方法中被调用，回忆下 `errors.go` 中的代码：

```go
func (f *fundamental) Format(s fmt.State, verb rune) {
    switch verb {
    case 'v':
        if s.Flag('+') {
            io.WriteString(s, f.msg)
            f.stack.Format(s, verb)
            return
        }
        fallthrough
    case 's':
        io.WriteString(s, f.msg)
    case 'q':
        fmt.Fprintf(s, "%q", f.msg)
    }
}
```

`f.stack.Format(s, verb)` 就是调用了 `Format` 方法，可以发现 `stack.Format` 方法只有在使用 `%+v` 时才生效，这也与我们之前总结的错误输出格式所对应。

在遇到 `%+v` 这个 `case` 时，程序会遍历 `stack` 切片中的每一项，即每一个栈帧，并且使用 `f := Frame(pc)` 将其包装成 `Frame` 类型，然后对其进行格式化输出 `fmt.Fprintf(st, "\n%+v", f)`。

所以我们要再看一下 `Frame` 是什么：

```go
// Frame represents a program counter inside a stack frame.
// For historical reasons if Frame is interpreted as a uintptr
// its value represents the program counter + 1.
type Frame uintptr
```

实际上 `Frame` 底层类型正是 `uintptr` 类型，所以可以直接通过 `Frame(pc)` 进行转换。

`Frame` 支持的方法定义如下：

```go
func (f Frame) pc() uintptr { return uintptr(f) - 1 }

func (f Frame) file() string {
    fn := runtime.FuncForPC(f.pc())
    if fn == nil {
        return "unknown"
    }
    file, _ := fn.FileLine(f.pc())
    return file
}

func (f Frame) line() int {
    fn := runtime.FuncForPC(f.pc())
    if fn == nil {
        return 0
    }
    _, line := fn.FileLine(f.pc())
    return line
}

func (f Frame) name() string {
    fn := runtime.FuncForPC(f.pc())
    if fn == nil {
        return "unknown"
    }
    return fn.Name()
}

// Format formats the frame according to the fmt.Formatter interface.
//
//  %s    source file
//  %d    source line
//  %n    function name
//  %v    equivalent to %s:%d
//
// Format accepts flags that alter the printing of some verbs, as follows:
//
//  %+s   function name and path of source file relative to the compile time
//        GOPATH separated by \n\t (<funcname>\n\t<path>)
//  %+v   equivalent to %+s:%d
func (f Frame) Format(s fmt.State, verb rune) {
    switch verb {
    case 's':
        switch {
        case s.Flag('+'):
            io.WriteString(s, f.name())
            io.WriteString(s, "\n\t")
            io.WriteString(s, f.file())
        default:
            io.WriteString(s, path.Base(f.file()))
        }
    case 'd':
        io.WriteString(s, strconv.Itoa(f.line()))
    case 'n':
        io.WriteString(s, funcname(f.name()))
    case 'v':
        f.Format(s, 's')
        io.WriteString(s, ":")
        f.Format(s, 'd')
    }
}

func (f Frame) MarshalText() ([]byte, error) {
    name := f.name()
    if name == "unknown" {
        return []byte(name), nil
    }
    return []byte(fmt.Sprintf("%s %s:%d", name, f.file(), f.line())), nil
}
```

我们重点关注 `Format` 方法，在 `stack.Format` 中执行 `fmt.Fprintf(st, "\n%+v", f)` 时就会调用 `Frame.Format` 方法。

结合注释我们很容易能总结出 `Frame` 类型输出格式有如下几种：

|verb|输出|
|---|---|
|%s|源码所在文件路径|
|%d|源码行号|
|%n|函数名称|
|%v|等价于 `%s:%d`|
|%+s|函数名称 + `\n\t` + 源码所在文件路径|
|%n|等价于 `%+s:%d`|

另外几个辅助函数我就不逐行解释了，我给你讲个示例，你就都明白了。

```go
package main

import (
    "fmt"
    "runtime"
    "strconv"
)

func printStack(skip int) {
    var pcs [30]uintptr
    n := runtime.Callers(skip, pcs[:])

    for i := 0; i < n; i++ {
        pc := pcs[i]
        fn := runtime.FuncForPC(pc - 1)
        file, line := fn.FileLine(pc - 1)
        fmt.Printf("Func Name: %s\n", fn.Name())
        fmt.Printf("File: %s, Line: %s\n\n", file, strconv.Itoa(line))
    }
}

func Print(skip int) {
    printStack(skip)
}

func main() {
    Print(0)

    fmt.Println("============================================")

    Print(3)
}
```

示例代码中，函数调用链为：`main() -> Print() -> printStack()`。

`printStack` 函数参数 `skip` 用来指定跳过的函数调用栈帧数量。`runtime.FuncForPC` 函数接收一个程序计数器 `pc` 值，然后返回 `*runtime.Func` 类型对象 `fn`，通过 `fn.FileLine(pc - 1)` 可以拿到文件路径和行号，`fn.Name()` 可以拿到函数名。

执行示例代码，输出结果如下：

```sh
$ go run main.go
Func Name: runtime.Callers
File: /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.22.7.darwin-arm64/src/runtime/extern.go, Line: 325

Func Name: main.printStack
File: /go/blog-go-example/error/pkg-errors/runtime-example/main.go, Line: 11

Func Name: main.Print
File: /go/blog-go-example/error/pkg-errors/runtime-example/main.go, Line: 23

Func Name: main.main
File: /go/blog-go-example/error/pkg-errors/runtime-example/main.go, Line: 27

Func Name: runtime.main
File: /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.22.7.darwin-arm64/src/runtime/proc.go, Line: 271

Func Name: runtime.goexit
File: /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.22.7.darwin-arm64/src/runtime/asm_arm64.s, Line: 1222

============================================
Func Name: main.main
File: /go/blog-go-example/error/pkg-errors/runtime-example/main.go, Line: 31

Func Name: runtime.main
File: /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.22.7.darwin-arm64/src/runtime/proc.go, Line: 271

Func Name: runtime.goexit
File: /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.22.7.darwin-arm64/src/runtime/asm_arm64.s, Line: 1222
```

可以看到，`Print(0)` 会从 `runtime.Callers` 栈帧开始输出，`Print(3)` 则会跳过 `runtime.Callers`、`main.printStack`、`main.Print` 这三个函数，从 `main.main` 开始输出。

你也许会好奇，`runtime.FuncForPC(pc - 1)` 和 `fn.FileLine(pc - 1)` 函数的调用参数为什么是 `pc - 1`，而不是 `pc`？

这个问题其实在 `Frame` 定义处的注释中有说明：

```go
// Frame represents a program counter inside a stack frame.
// For historical reasons if Frame is interpreted as a uintptr
// its value represents the program counter + 1.
type Frame uintptr
```

但看完了注释就更加疑惑了，由于历史原因 `runtime.Callers` 返回值是程序计数器 + 1，历史原因又是什么呢？

我们可以深入到 `runtime.Callers` 源码，顺着 `runtime.Callers() -> callers() -> tracebackPCs()` 调用链路，一路跟踪到 `tracebackPCs` 函数的定义中：

```go
func tracebackPCs(u *unwinder, skip int, pcBuf []uintptr) int {
    var cgoBuf [32]uintptr
    n := 0
    for ; n < len(pcBuf) && u.valid(); u.next() {
        f := u.frame.fn
        cgoN := u.cgoCallers(cgoBuf[:])

        // TODO: Why does &u.cache cause u to escape? (Same in traceback2)
        for iu, uf := newInlineUnwinder(f, u.symPC()); n < len(pcBuf) && uf.valid(); uf = iu.next(uf) {
            sf := iu.srcFunc(uf)
            if sf.funcID == abi.FuncIDWrapper && elideWrapperCalling(u.calleeFuncID) {
                // ignore wrappers
            } else if skip > 0 {
                skip--
            } else {
                // Callers expect the pc buffer to contain return addresses
                // and do the -1 themselves, so we add 1 to the call PC to             // create a return PC.             pcBuf[n] = uf.pc + 1
                n++
            }
            u.calleeFuncID = sf.funcID
        }
        // Add cgo frames (if we're done skipping over the requested number of
        // Go frames).       if skip == 0 {
        n += copy(pcBuf[n:], cgoBuf[:cgoN])
        }
    }
    return n
}
```

这里也有一段注释：

```go
// Callers expect the pc buffer to contain return addresses
// and do the -1 themselves, so we add 1 to the call PC to
// create a return PC.
```

可以简单理解为，调用方期望拿到 `pc` 并自行执行 `-1` 操作，因此 `runtime.Callers` 函数内部会在调用时为 `pc` 加 `1` 来生成返回地址。

不得不吐槽下这个逻辑隐藏的太深了，不深入源码看根本不知道其意图。我猜测可能会在 Go 仓库的某个历史 issues 中有一些说明，如果你感兴趣可以去翻一翻。

> NOTE: 其实 `runtime` 包提供了另外一个 `CallersFrames` 方法，也能过获取函数名称、行号等，与 `runtime.Callers` 搭配使用更佳方便，无需考虑 `pc - 1` 问题，你可以去我的 GitHub 仓库查看我实现的示例代码：https://github.com/jianghushinian/blog-go-example/blob/main/error/pkg-errors/runtime-example/main.go#L34。

扯远了，最后我们再回过头来看看 `stack.StackTrace` 方法的逻辑：

```go
func (s *stack) StackTrace() StackTrace {
    f := make([]Frame, len(*s))
    for i := 0; i < len(f); i++ {
        f[i] = Frame((*s)[i])
    }
    return f
}
```

可以看到，这里就是在组装 `[]Frame` 切片，然后返回。

返回类型 `StackTrace` 定义如下：

```go
// StackTrace is stack of Frames from innermost (newest) to outermost (oldest).
type StackTrace []Frame
```

它有两个方法：

```go
// Format formats the stack of Frames according to the fmt.Formatter interface.
//  %s lists source files for each Frame in the stack
//  %v lists the source file and line number for each Frame in the stack
//
// Format accepts flags that alter the printing of some verbs, as follows:
//
//  %+v   Prints filename, function, and line number for each Frame in the stack.
func (st StackTrace) Format(s fmt.State, verb rune) {
    switch verb {
    case 'v':
        switch {
        case s.Flag('+'):
            for _, f := range st {
                io.WriteString(s, "\n")
                f.Format(s, verb)
            }
        case s.Flag('#'):
            fmt.Fprintf(s, "%#v", []Frame(st))
        default:
            st.formatSlice(s, verb)
        }
    case 's':
        st.formatSlice(s, verb)
    }
}

func (st StackTrace) formatSlice(s fmt.State, verb rune) {
    io.WriteString(s, "[")
    for i, f := range st {
        if i > 0 {
            io.WriteString(s, " ")
        }
        f.Format(s, verb)
    }
    io.WriteString(s, "]")
}
```

经过前面的源码分析，相信这两个方法你自己也能看懂了，就留给你自己分析吧。

至此，`pkg/errors` 包的源码就全部解读完成了。

现在的你已经掌握了 `pkg/errors` 包是如何设计的，它的更多使用技巧就等着你自行去探索了，我就不一一举例了。

### 总结

本文从 Go 错误处理常见的两个诉求说起，Go 自带的错误处理无法很好的处理附加错误信息和附加堆栈信息。而 `pkg/errors` 包则完全能够满足我们的诉求，并且有很好的兼容性。

我带你分析了 `pkg/errors` 包的全部源码，`pkg/errors` 包含三个文件：

- `errors.go` 定义了三种错误类型 `fundamental`、`withStack`、`withMessage` 供我们使用，借助这三种错误类型，我们可以很方便的组装 `error`、`msg` 和 `stack` 这些信息。
- `go113.go` 是为了兼容 Go 1.13 版本而引入的文件，分别对 `Is`、`As`、`Unwrap` 这三个函数进行了代理。
- `stack.go` 中的代码用来处理错误堆栈信息。通过此文件我们还学习到了 `runtime.Callers` 和 `runtime.FuncForPC` 的用法。

这也是我写「Go 错误处理指北」系列的第二篇文章，接下来还会有更多的文章继续讲解 Go 中的错误处理，敬请期待。

本文示例源码我都放在了 GitHub 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- pkg/errors Documentation：https://pkg.go.dev/github.com/pkg/errors
- pkg/errors 源码：https://github.com/pkg/errors/
- Go 1.13 Release Notes：https://tip.golang.org/doc/go1.13#error_wrapping
- Errors are values：https://go.dev/blog/errors-are-values
- Go 错误处理指北：Error vs Exception vs ErrNo：https://jianghushinian.cn/2024/09/06/go-error-guidelines-error-exception-errno/
- 如何规范 RESTful API 的业务错误处理：https://jianghushinian.cn/2023/03/04/how-to-standardize-the-handling-of-restful-api-business-errors/
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/error/pkg-errors
