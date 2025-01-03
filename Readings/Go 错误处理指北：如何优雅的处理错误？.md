---
title: Go 错误处理指北：如何优雅的处理错误？
author:
  - Go 编程世界
date created: 2025-01-01 14:35:10
category: Golang
tags:
  - "#Golang"
  - "#编程实践"
  - "#错误处理"
  - "#软件开发"
  - "#代码优化"
url:
  - https://mp.weixin.qq.com/s/ImvwsAUhQ3MMZkKvnbNB3A
description: 这篇文章深入探讨了如何在 Go 语言中优雅地处理错误。文章首先对比了 Go 与其他编程语言的错误处理机制，解释了 Go 为什么选择 Error 而非异常机制。接着，文章详细介绍了如何构造和处理错误，包括 Sentinel error、Opaque error、类型断言、行为断言等多种处理方式。此外，文章还强调了错误处理的最佳实践，如避免冗余错误检查、确保错误只处理一次，以及在日志记录中确保错误存在等。最后，文章提供了丰富的延伸阅读资源，帮助读者进一步理解和应用 Go 的错误处理机制。
status: Finished
---

本文是 **Go 错误处理指北**系列第三篇文章：如何优雅的处理错误？

作为铺垫，我在系列的前两篇文章 [Go 错误处理指北：Error vs Exception vs ErrNo](Go%20错误处理指北：Error%20vs%20Exception%20vs%20ErrNo.md) 和 [Go 错误处理指北：pkg／errors 源码解读](Go%20错误处理指北：pkg／errors%20源码解读.md) 中分别讲解了 Go 错误处理机制和流行的第三方包 `pkg/errors`，现在是时候对 Go 语言中的错误处理做一个比较全面的讲解了。

### Go 中为什么没有 Exception

我在 [Go 错误处理指北：Error vs Exception vs ErrNo](Go%20错误处理指北：Error%20vs%20Exception%20vs%20ErrNo.md) 一文中对比过 Python、C、Go 这三种编程语言错误处理的不同之处。其中 Python 的 `Exception` 异常处理机制是主流编程语言中最为流行的方式，可是 Go 为什么采用了 `Error` 机制呢？

Go 官方的 FAQ: Why does Go not have exceptions? 中给出了解释：

> 我们认为，将异常与控制结构耦合在一起（如 `try-catch-finally` 语句）会导致代码变得复杂。同时，这也往往会促使程序员将太多普通的错误（比如打开文件失败）标记为异常。
>
> Go 采用了一种不同的处理方式。对于普通的错误处理，Go 函数支持多返回值机制使得在不覆盖返回值的情况下，能够轻松地报告错误。Go 还提供了一个标准的错误类型，再加上其他特性，使得错误处理变得简洁而又与其他语言截然不同。
>
> Go 还提供了一些内置函数，用于标识和恢复真正的异常情况。恢复机制只会在函数状态因错误而被销毁时执行，这足以处理灾难性错误，同时不需要额外的控制结构。使用得当时，可以写出简洁的错误处理代码。
>
> 详情请参考 Defer, Panic, and Recover 一文。另外，博客文章 Errors are values 展示了一种整洁的错误处理方式，说明了由于错误只是值，Go 语言的全部能力都可以用于处理错误。

说白了，Go 官方认为 `Error` 机制更简单有效，且符合 Go 语言大道至简的调性。

### 构造错误

既然要讲解如何处理错误，那么就先从如何构造一个错误说起吧。

我们知道，Go 的 `error` 实际上就是一个普通的接口，普普通通：

```go
type error interface {
    Error() string
}
```

得益于 Go 函数支持多返回值的能力，我们可以非常方便的返回一个错误：

```go
func foo() (string, error) {
    // do something
    return "", nil
}
```

> NOTE: 当函数返回多个值时，`error` 作为最后一个返回值是约定俗成的惯用法。如果你不这么做，代码当然能成功编译，但你有更好的选择。

Go 提供了两种构造错误的方式：

```go
// 创建一个错误值
err1 := errors.New("example err1")
// 格式化错误消息
err2 := fmt.Errorf("example err2: %d", userID)
```

这两种构造错误的方式最终都是返回 `errorString` 类型的指针：

```go
// errors.New 函数定义
func New(text string) error {
    return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

> NOTE: 其实 `fmt.Errorf` 内部也是调用 `errors.New` 来创建 `error`。当然，在 Go 1.13 版本以后，`fmt.Errorf` 可能会在特定条件下返回 `wrapError` 类型错误。

### 处理错误

现在我们已经可以构造一个错误，接下来看看如何优雅的处理错误。

#### 错误处理惯用法

如下示例是 Go 中经典的错误处理方式：

```go
data, err := foo()
if err != nil {
    // 处理错误
    return
}
// 正常逻辑
fmt.Println(data)
```

一切的错误处理都从 `if err != nil` 开始。

#### Sentinel error

预定义的错误值：`Sentinel error`，一般被译为 `哨兵错误`。这是一种错误处理惯用法，在 Go 内置包中有大量应用。

比如：

> https://github.com/golang/go/blob/go1.23.1/src/bufio/bufio.go#L22

```go
var (
    ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
    ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
    ErrBufferFull        = errors.New("bufio: buffer full")
    ErrNegativeCount     = errors.New("bufio: negative count")
)
```

或者：

> https://github.com/golang/go/blob/go1.23.1/src/io/io.go#L29

```go
var ErrShortWrite = errors.New("short write")
var errInvalidWrite = errors.New("invalid write result")
var ErrShortBuffer = errors.New("short buffer")
var EOF = errors.New("EOF")
var ErrUnexpectedEOF = errors.New("unexpected EOF")
var ErrNoProgress = errors.New("multiple Read calls return no data or error")
```

这些都叫 `Sentinel error`，绝大多数 `Sentinel error` 都会被定义为包级别公开变量，可以看到也有内置的 `errInvalidWrite` 并没有对外公开。 ^sentinelError

每个 `error` 变量都以前缀 `Err` 开头，这是约定俗成的做法。`io.EOF` 是个特例，因为 EOF 是另一种约定用法，它的全拼是 `end of file`，表示文件结束，应用非常广泛，可以算作专有名词了。

我们可以像这样处理 `Sentinel error`：

```go
if err != nil {
    if err == bufio.ErrBufferFull {
        // handle ErrBufferFull
    }
    // handle err
}
```

要处理多种错误类型时，可以使用 `switch…case…` 语句来简化处理：

```go
f, err := os.Open("example.txt")
if err != nil {
    return
}

b := bufio.NewReader(f)

data, err := b.Peek(10)
if err != nil {
    switch err {
    case bufio.ErrNegativeCount:
        // do something
        return
    case bufio.ErrBufferFull:
        // do something
        return
    default:
        // do something
        return
    }
}
fmt.Println(string(data))
```

示例中 `b.Peek(10)` 可能会返回 `ErrNegativeCount` 或 `ErrBufferFull` 错误变量，因为它们是依赖包中可导出的公开变量，所以我们可以在自己的代码中使用这些变量来识别返回了哪个特定的错误消息。

也就是说，这些 `Sentinel error`  **变量会成为包 API 的一部分**，用于错误处理。

如果没有 `Sentinel error` 的存在，我们可能需要通过字符串匹配的方式来识别错误类型：

```go
if err != nil {
    if strings.Contains(err.Error(), "buffer full") {
        // Processing
    }
}
```

**我个人完全不赞成这种写法，不到万不得已，千万不要写成这种代码。**

**记住：`error` 接口上的 `Error` 方法适用于人类，而非代码**。只有我们需要查看错误信息，或者记录日志的时候，才应该使用 `Error` 方法。

此外，你可能在标准库中见到过如下类似代码：

> https://github.com/golang/go/blob/go1.23.1/src/os/error.go#L16

```go
var (
    // ErrInvalid indicates an invalid argument.
    // Methods on File will return this error when the receiver is nil.
    ErrInvalid = fs.ErrInvalid // "invalid argument"

    ErrPermission = fs.ErrPermission // "permission denied"
    ErrExist      = fs.ErrExist      // "file already exists"
    ErrNotExist   = fs.ErrNotExist   // "file does not exist"
    ErrClosed     = fs.ErrClosed     // "file already closed"
)
```

`os.ErrInvalid` 实际上等价于 `fs.ErrInvalid`，这种为 `Sentinel error` 重新赋值的操作也很常见。为了保持良好的分层架构，我们自己的代码设计也可以这样做。

另外，`Sentinel error` 还有一种看似 “另类” 的用法，表示错误没有发生，比如 `path/filepath.SkipDir`：

> https://github.com/golang/go/blob/go1.23.1/src/path/filepath/path.go#L259

```go
// SkipDir is used as a return value from [WalkDirFunc] to indicate that
// the directory named in the call is to be skipped. It is not returned
// as an error by any function.
// // SkipDir 用作 [WalkDirFunc] 的返回值，表示要跳过调用中指定的目录。任何函数都不会将其作为错误返回。
var SkipDir = errors.New("skip this directory")
```

根据注释我们可以了解到，`SkipDir` 变量用作 `WalkDirFunc` 的返回值，以指示将跳过调用中指定的目录，它并不表示一个错误。

所以这里 `SkipDir` 仅作为哨兵，而非错误。其实 `io.EOF` 也是哨兵，并且它们都没有以 `Err` 来命名。

这也是我认为 `Sentinel error` 存在二义性的地方，我个人认为绝大多数情况下不应该这么使用，尽量避免这种用法。

#### 常量错误

因为 `Sentinel error` 是一个变量，所以我们可以随意改变它的值：

```go
oldEOF := io.EOF
io.EOF = errors.New("MyEOF")
fmt.Println(oldEOF == io.EOF)   // false
```

这是一个很可怕的事情。

所以 `Sentinel error` 的确不是一个好的设计，起码也应该将其定义成一个常量。

但问题是在 Go 中我们无法直接将 `errors.New` 的返回值赋值给一个常量。

如下示例：

```go
const ErrMyEOF = errors.New("MyEOF")
```

这将得到编译报错：

```sh
errors.New("MyEOF") (value of type error) is not constant
```

为了解决这个问题，我们可以自定义 `error` 类型：

```go
type Error string

func (e Error) Error() string { return string(e) }
```

`Error` 类型底层类型为 `string`，所以可以直接赋值给一个常量：

```go
const ErrMyEOF = Error("MyEOF")
```

现在常量 `ErrMyEOF` 不可改变。

但是，这又会引入另外一个新的问题。以下示例代码，执行结果为 `true`：

```go
const ErrMyEOF = Error("MyEOF")
const ErrNewMyEOF = Error("MyEOF")
fmt.Println(ErrMyEOF == ErrNewMyEOF)    // true
```

这与 Go 内置的 `errors.New` 表现并不相同。

以下示例代码，执行结果为 `false`：

```go
myEOF = errors.New("EOF")
fmt.Println(io.EOF == myEOF)    // false
```

造成二者表现不同的原因是：内置的 `errors.New` 函数返回 `errorString` 的指针类型 `&errorString{text}`，而我们构造的自定义 `Error` 实际上是 `string` 类型。

`errors.New` 返回指针类型是有意而为之的，目的就是在判断两个错误值是否相等时，会比较两个对象是否为同一个对象，而不是比较 `Error` 方法所返回的字符串内容是否相等。如果仅比较字符串内容是否相等，则我们随便使用 `errors.New` 函数创建的错误就可以实现与预置的 `Sentinel error` 相等。

所以常量错误并不常见，我个人其实也不太推荐一定要追求把错误定义为常量，适当引入的编码规范更加切合实际。

尽管 `errorString` 类型仅包含一个字段 `s string`，但它还是被有意设计成 `struct` 而非简单的 `string` 类型别名，否则 `Sentinel error` 实用价值将大大折扣。

#### 定制错误类型

与使用 `errors.New` 创建出来的 `*errorString` 错误值相比，定制错误类型往往能提供更多的上下文信息。

Go 内置库中就有这样的例子，比如错误类型 `os.PathError`：

> https://github.com/golang/go/blob/go1.23.1/src/io/fs/fs.go#L250

```go
// PathError records an error and the operation and file path that caused it.
type PathError struct {
    Op   string
    Path string
    Err  error
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
```

> NOTE: 错误类型命名通常以 `Error` 结尾，这是约定俗成的惯用法。

`PathError` 类型不仅能够记录错误，还会记录导致出现错误的操作和文件路径。在出现错误时，更方便排查问题。

有了新的错误类型后，最大的好处是可以通过类型断言，来判断错误的类型。如果断言成立，则可以根据错误类型对当前错误做更为精细的控制。

示例如下：

```go
// 尝试打开一个不存在的文件
_, err := os.Open("nonexistent.txt")
if err != nil {
    // 使用类型断言检查是否为 *os.PathError 类型
    if pathErr, ok := err.(*os.PathError); ok {
        fmt.Printf("Failed to %s file: %s\n", pathErr.Op, pathErr.Path)
        fmt.Println("Error message:", pathErr.Err)
    } else {
        // 其他类型的错误处理
        fmt.Println("Error:", err)
    }
}
```

可以发现，为了实现错误类型的断言检查，`PathError` 类型必须是公开类型。

其实无论是 `Sentinel error`，还是自定义的错误类型，它们都存在同样的问题，都会成为包 API 的一部分，被公开出去。这很可能导致包 API 的快速膨胀。并且，如果代码分层设计不好，很容易出现循环依赖问题。

#### Opaque error

`Opaque error` 是 Go 语言布道师 Dave Cheney 在 Gocon Spring 2016 演讲中提出的一种叫法，姑且把它翻译为 `不透明的错误处理`。

`Opaque error` 非常简单，它是最灵活的错误处理策略，因为它需要代码和调用者之间的耦合最少。

示例如下：

```go
func fn() error {
    x, err := bar.Foo()
    if err != nil {
        return err
    }
    // use x
}
```

这就是 `Opaque error` 的全部内容了：只需返回错误，而不对其内容做出任何假设。

没错，遇到错误后直接 `return err` 的做法就是 `Opaque error`。

显然，这种代码看似优雅，却过于理想。现实中我们仍有很多情况下还是需要知道错误内容，然后决定是否对其进行处理。

#### 错误值比较

比较两个错误值是否相等的操作，一般结合 `Sentinel error` 一同使用：

```go
if err != nil {
    if err == bufio.ErrBufferFull {
        // handle ErrBufferFull
    }
    // handle err
}
```

先使用 `if err != nil` 与 `nil` 比较来判定是否存在错误，如果有错误，更进一步，使用 `if err == bufio.ErrBufferFull` 来判定错误是否为某个 `Sentinel error`。

当可能出现多种错误时，还可以使用 `switch…case…` 来判定错误值：

```go
if err != nil {
    switch err {
    case bufio.ErrNegativeCount:
        // do something
        return
    case bufio.ErrBufferFull:
        // do something
        return
    default:
        // do something
        return
    }
}
```

#### 类型断言

Go 支持两种类型断言，Type Assertion 和 Type Switch。

Go 的类型断言语法可以直接应用于错误处理，因为 `error` 本身就是一个普通的接口。

断言一个错误的类型，其实前文中我们已经见过了：

```go
// 尝试打开一个不存在的文件
_, err := os.Open("nonexistent.txt")
if err != nil {
    // 使用类型断言检查是否为 *os.PathError 类型
    if pathErr, ok := err.(*os.PathError); ok {
        fmt.Printf("Failed to %s file: %s\n", pathErr.Op, pathErr.Path)
        fmt.Println("Error message:", pathErr.Err)
    } else {
        // 其他类型的错误处理
        fmt.Println("Error:", err)
    }
}
```

如果改用 `switch…case…` 可以这样写：

```go
// 尝试打开一个不存在的文件
_, err := os.Open("nonexistent.txt")
if err != nil {
    // 使用 switch type 检查错误类型
    switch e := err.(type) {
    case *os.PathError:
        fmt.Printf("Failed to %s file: %s\n", e.Op, e.Path)
        fmt.Println("Error message:", e.Err)
    default:
        // 其他类型的错误处理
        fmt.Println("Error:", err)
    }
}
```

值得一提的是，在使用 `Type Switch` 语法时，是禁止使用 `fallthrough` 关键字的，否则编译报错 `cannot fallthrough in type switch`。

> 在 Go 语言中，`fallthrough` 关键字用于在 `switch` 语句中强制执行下一个 case 的代码块。通常，`switch` 中的每个 case 执行后会自动退出，但使用 `fallthrough` 可以让程序继续执行紧接着的下一个 case，而不进行条件判断。（说白了就是开启 C/C++ 模式的 swtich-case）
>
> 需要注意的是：
>
> 1. `fallthrough` 只能用于 `switch` 语句中。
> 2. `fallthrough` 必须是 case 语句中的最后一条语句。
> 3. 它会直接执行下一个 case 的代码，而不会检查下一个 case 的条件。
>
>示例：
>
> switch x {
> case 1:
>     fmt.Println("Case 1")
>     fallthrough
> case 2:
>     fmt.Println("Case 2")
> case 3:
>     fmt.Println("Case 3")
> }
>
>如果 `x` 等于 1，输出将是：
>
> Case 1
> Case 2
>
> 即使 `x` 并不等于 2，`fallthrough` 仍然会使程序继续执行 case 2 的代码。

这种情况 `case` 语句只能使用逗号并提供多个选项：

```go
if err != nil {
    switch err.(type) {
    case *os.PathError, *os.LinkError:
        // do something
    default:
        // do something
    }
}
```

这两种方法的最大缺点就是我们需要导入指定的错误类型，如示例中的 `os.PathError` 或 `os.LinkError`。这会导致我们的代码与错误所在的包存在较强的依赖关系。

#### 行为断言

随着 Go 语言的演进，大家对 Go 的错误处理又有了新的理解。以前断言错误类型，现在社区中则更推荐断言错误行为。

Go 语言布道师 Dave Cheney 在他的文章 Inspecting errors 中提出了**断言错误行为而不是类型**。

> NOTE: 没错，`Dave Cheney` 的名字再一次出现，后文还会出现😄。这位大佬对 Go 社区的贡献很大，尤其是错误处理，著名的 `pkg/errors` 包就是他开发的。

```go
func isTimeout(err error) bool {
    type timeout interface {
        Timeout() bool
    }
    te, ok := err.(timeout)
    return ok && te.Timeout()
}
```

函数 `isTimeout` 用来判定一个错误对象是否表示 `Timeout`，内部通过断言错误对象是否实现了 `timeout` 接口来实现。

我们不再假设错误的类型，而是假设其实现了某个接口，并且 `timeout` 接口是一个临时接口，并不是从其他包中导入的接口类型。这样就真正的实现了包之间的解耦，错误类型无需公开，它们不再必须是包 API 的一部分。

`net.Error` 就是一个比较不错的实践：

> https://github.com/golang/go/blob/go1.23.1/src/net/net.go#L415

```go
// An Error represents a network error.
type Error interface {
    error
    Timeout() bool  // Is the error a timeout?

    // Deprecated: Temporary errors are not well-defined.
    // Most "temporary" errors are timeouts, and the few exceptions are surprising.
    // Do not use this method.
    Temporary() bool
}
```

客户端代码可以断言错误是否为 `net.Error` 类型，然后再根据行为区分暂时性网络错误和永久性网络错误。

例如，一个爬虫程序在遇到临时错误时可以短暂休眠并重试，否则放弃这个请求，直接处理错误。

示例代码如下：

```go
if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
    time.Sleep(1e9)
    continue
}
if err != nil {
    log.Fatal(err)
}
```

当然，这段代码还可以写成这样：

```go
if nerr, ok := err.(interface{
    Temporary() bool
}); ok {
    time.Sleep(1e9)
    continue
}
if err != nil {
    log.Fatal(err)
}
```

这样就实现了我们的代码与错误所在的包之间最大化的解耦。

当然这两种写法其实都可以，看个人喜好。

#### 暂存错误状态

在使用 `Builder 模式`、链式调用或者 `for` 循环等场景下，暂存中间过程所出现的错误，有助于简化代码，使编写出的代码逻辑更加连贯。

> NOTE: 如果你不了解 `Builder 模式`，可以查阅我的另一篇文章：[Builder 模式在 Go 语言中的应用](Builder%20模式在%20Go%20语言中的应用.md)。

以下示例中使用 K8s `client-go` SDK 提供的 `clientset` 客户端查询 `pod` 信息：

```go
pod, err := clientset.CoreV1().Pods("default").Get(ctx, "nginx", metav1.GetOptions{})
if err != nil {
    // do something
}
```

其中的 `Get` 方法内部，就使用了链式调用，源码如下：

> https://github.com/kubernetes/kubernetes/blob/release-1.30/staging/src/k8s.io/client-go/kubernetes/typed/core/v1/pod.go#L75

```go
// Get takes name of the pod, and returns the corresponding pod object, and an error if there is any.
func (c *pods) Get(ctx context.Context, name string, options metav1.GetOptions) (result *v1.Pod, err error) {
    result = &v1.Pod{}
    err = c.client.Get().
        Namespace(c.ns).
        Resource("pods").
        Name(name).
        VersionedParams(&options, scheme.ParameterCodec).
        Do(ctx).
        Into(result)
    return
}
```

`c.client.Get()` 会返回一个 `*Request` 对象，接着调用它的 `Namespace(c.ns)` 方法：

> https://github.com/kubernetes/kubernetes/blob/release-1.30/staging/src/k8s.io/client-go/rest/request.go#L294

```go
// Namespace applies the namespace scope to a request (<resource>/[ns/<namespace>/]<name>)
func (r *Request) Namespace(namespace string) *Request {
    if r.err != nil {
        return r
    }
    if r.namespaceSet {
        r.err = fmt.Errorf("namespace already set to %q, cannot change to %q", r.namespace, namespace)
        return r
    }
    if msgs := IsValidPathSegmentName(namespace); len(msgs) != 0 {
        r.err = fmt.Errorf("invalid namespace %q: %v", namespace, msgs)
        return r
    }
    r.namespaceSet = true
    r.namespace = namespace
    return r
}
```

`*Request.Namespace` 方法首先会通过 `if r.err != nil` 判断是否存在错误，如果存在则直接返回，不再继续执行。如果不存在错误，则接下来每次可能出现错误的调用，都会将错误信息暂存到 `r.err` 属性中。

接下来是调用 `Resource("pods")` 方法：

> https://github.com/kubernetes/kubernetes/blob/release-1.30/staging/src/k8s.io/client-go/rest/request.go#L210

```go
// Resource sets the resource to access (<resource>/[ns/<namespace>/]<name>)
func (r *Request) Resource(resource string) *Request {
    if r.err != nil {
        return r
    }
    if len(r.resource) != 0 {
        r.err = fmt.Errorf("resource already set to %q, cannot change to %q", r.resource, resource)
        return r
    }
    if msgs := IsValidPathSegmentName(resource); len(msgs) != 0 {
        r.err = fmt.Errorf("invalid resource %q: %v", resource, msgs)
        return r
    }
    r.resource = resource
    return r
}
```

`*Request.Resource` 方法内部代码逻辑的套路，与 `*Request.Namespace` 方法如出一辙。

`client-go` 就是通过将错误暂存到 `*Request.err` 属性的方式，简化了使用侧的代码逻辑。我们可以放心编写代码中的链式调用，只在最后处理一次错误即可。如果调用链中间某个方法出现了错误，之后执行的方法都能够自行处理。

#### 返回错误而不是指针

Dave Cheney 在他的文章 Errors and Exceptions, redux 中列举了一个程序示例：

```go
func Positive(n int) (bool, bool) {
    if n == 0 {
        return false, false
    }
    return n > -1, true
}
```

这是一个判断给定变量 `n` 的值为正负数的小函数。

`0` 既不是正数也不是负数，因此为了判断传进来的 `n` 是否为 `0`，函数必须返回两个值，第一个 `bool` 值标识 `正/负`，第二个 `bool` 值标识返回的第一个值是否有效，即 `n` 是否为 `0`。

还有一种实现方式是下面这样：

```go
func Positive(n int) (bool, error) {
    if n == 0 {
        return false, errors.New("undefined")
    }
    return n > -1, nil
}
```

我们使用 `error` 来作为 `Sentinel`，标识 `n` 是否为 `0`。

以上两种方式我个人认为都可以接受，看个人喜好选择即可。

不过，有人可能会有不同的实现：

```go
func Positive(n int) *bool {
    if n == 0 {
        return nil
    }
    r := n > -1
    return &r
}
```

这次实现的 `Positive` 函数仅有一个返回值，类型是 `*bool`。

当 `*bool` 值为 `nil`，标识 `n` 是否为 `0`。

当 `*bool` 值不为 `nil`，其解引用后，值为 `true` 标识结果为 `正`，值为 `false` 标识结果为 `负`。

这种做法极其不推荐，不仅使返回值存在二义性，还使调用方代码变得啰嗦。在任何地方使用返回值之前，我们都必须检查它以确保它指向的地址有效。

#### Errors are values

Errors are values 是 Rob Pike 提出来的，旨在纠正人们对 Go 错误处理的认知。

`Errors are values` —— **错误就是值**！它没什么特殊的，你在使用 Go 语言编程过程中，可以像对待其他任何普通类型一样对待错误。

所以，我们可以对错误进行等值比较、类型断言、行为断言等操作。遗憾的是，这是非常基本的东西，大多数 Go 程序员却没有注意到。

现在我们有如下示例代码：

```go
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}

_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}

_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// and so on
```

这里存在非常严重的重复，这也是 `if err != nil` 容易被吐槽的典型场景。

不过，我们可以编写一个简单的辅助函数，来解决这个问题：

```go
var err error
write := func(buf []byte) {
    if err != nil {
        return
    }
    _, err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])
// and so on
if err != nil {
    return err
}
```

现在，代码看起来是不是好了一些，没有了那么多重复的 `if err != nil`。

我们还可以对这个示例程序做进一步优化：

```go
type errWriter struct {
    w   io.Writer
    err error
}

func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

定义一个 `errWriter` 结构体，来代理 `io.Writer` 对象的写操作，并能暂存错误。

可以这样使用 `errWriter`：

```go
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// and so on
if ew.err != nil {
    return ew.err
}
```

现在的代码使用辅助结构体更为优雅的解决了 `if err != nil` 重复问题。

这里正是采用了前文中讲解的 暂存错误状态 思想。

之所以列举这个示例，就是为了告诉大家，你可以像对待其他任何普通类型一样对待错误。错误可以作为 `errWriter` 结构体的一个属性存在，这没什么不妥，像编写其他代码一样正常的处理错误即可。

记住：**错误就是值**。

#### 不要忽略你的错误

任何时候不要写出这种代码：

```go
data, _ := Foo()
```

如果你确信 `Foo()` 的确不会返回错误，可以对其进行包装，在内部处理错误：

```go
func MustFoo() string {
    data, err := Foo()
    if err != nil {
        panic(err)
    }
    return data
}
```

现在，我们可以不关心错误直接调用 `MustFoo()`：

```go
data := MustFoo()
```

`MustXxx` 这种做法也算比较常见，比如 Gin 框架中就有很多这种风格的实现：

> https://github.com/gin-gonic/gin/blob/v1.10.0/context.go#L280

```go
// Get returns the value for the given key, ie: (value, true).
// If the value does not exist it returns (nil, false)
func (c *Context) Get(key string) (value any, exists bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    value, exists = c.Keys[key]
    return
}

// MustGet returns the value for the given key if it exists, otherwise it panics.
func (c *Context) MustGet(key string) any {
    if value, exists := c.Get(key); exists {
        return value
    }
    panic("Key \"" + key + "\" does not exist")
}
```

切记，不要忽略你的错误，除非这你把它当作一个 demo 程序，而非生产项目。

#### 错误只应处理一次

虽然，不应该忽略你的错误，但错误也不应该被重复处理，你应该只处理一次错误。

处理错误意味着检查错误值并做出决定。也就是说，根据能否处理错误，我们实际上只有两种选择：

1. 不能处理：直接向上传递错误，自身不对错误做任何假设，也就是 `Opaque error`。
2. 可以处理：降级处理，并向上返回 `nil`，因为自己已经处理了错误，表示不再有错误了，上层不应该继续拿到这个错误。

有如下示例：

```go
func Foo() error {
    return errors.New("foo error")
}

func Bar() error {
    err := Foo()
    if err != nil {
        // do something
        return err
    }
    return err
}

func main() {
    err := Bar()
    if err != nil {
        // do something
    }
}
```

在函数调用链中，错误被处理了两次。这是非常糟糕的实践，如果两处都记录日志，那么最终看到的日志将会有非常多的重复内容。

正确的做法应该是这样：

```go
func Foo () error {
    return errors.New("foo error")
}

func Bar() error {
    return Foo()
}

func main() {
    err := Bar()
    if err != nil {
        // do something
    }
}
```

仅在调用链顶端 `main` 函数中处理一次错误。

有时候，我们也可以这样处理：

```go
func Foo() error {
    return errors.New("foo error")
}

func Bar() error {
    err := Foo()
    if err != nil {
        // do something
        // NOTE: 这里提供服务降级处理，如记录日志
        return nil
    }
    // do something
    return nil
}

func main() {
    err := Bar()
    if err != nil {
        // do something
    }
}
```

这次，`Bar` 函数中遇到调用 `Foo()` 出错的情况，会进行降级，然后返回 `nil`。

错误同样只会被处理一次，`main` 函数永远也得不到 `Foo()` 函数返回的错误。

#### 审查你的错误处理代码

一些代码细节，也能让处理错误的代码更加优雅。

##### 在缩进中处理错误

在缩进中编写你的处理错误逻辑：

```go
f, err := os.Open(filename)
if err != nil {
    // handle error
}
// normal logic
```

而不是在缩进中处理正常逻辑：

```go
f, err := os.Open(filename)  
if err == nil {  
    // normal logic  
}  
// handle error
```

这样在 IDE 中能够非常方便的折叠 `if err != nil` 逻辑，使得阅读正常逻辑的代码更加清晰。

##### 不要做冗余的错误检查

不要写出这种代码：

```go
func Foo() error {
    err := Bar()
    if err != nil {
        return err
    }
    return nil
}
```

这里的错误检查完全没有任何意义，属于冗余代码。除非你的工作以代码行数来做 KPI 考核，否则没有任何理由这样写。

正确写法如下：

```go
func Foo() error {
    return Bar()
}
```

参考社区中的 Go 代码规范，更多代码细节就等着你自己去发现了。

#### nil 错误值可能不等于 nil

这个问题来自 Go 的 FAQ: Why is my nil error value not equal to nil ? 是我们开发过程中要注意的一个点。

有如下示例代码：

```go
package main

import "fmt"

type MyError struct {
    msg string
}

func (e *MyError) Error() string {
    return e.msg
}

func returnsError() error {
    var p *MyError = nil
    return p    // Will always return a non-nil error.
}

func main() {
    err := returnsError()
    if err != nil {
        fmt.Println("err:", err)
    return
    }
    fmt.Println("success")
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
err: <nil>
```

可以发现，`main` 函数中的 `if err != nil` 错误检查结果为 `true`，但使用 `fmt.Println` 输出的值却为 `nil`。

出现这一怪异现象的原因与 Go 的接口实现有关。

在 Go 中一个接口对象实际包含两个属性：类型 `T` 和具体的值 `V`。例如，如果我们将值为 `3` 的 `int` 类型对象存储在接口中，则生成的接口对象实际上内部保存了：`T=int, V=3`。

仅当 `T` 和 `V` 都未设置时（`T=nil, V 未设置`），接口的值才为 `nil`。

如果我们将 `*int` 类型的 `nil` 指针存储在接口对象中，则无论指针的值是什么，接口类型都将是 `T=*int, V=nil`。

同理虽然 `p` 在初始化时赋值为 `nil`（`var p *MyError = nil`），但是它会被赋值给接口类型 `error`，我们得到的接口类型将是 `T=*MyError, V=nil`。

所以，我们应该避免写出这种代码。

#### 错误与日志

错误与日志密不可分，在程序出错时记录日志能够方便我们排查问题。

##### 显式胜于隐式

我们知道 `fmt.Printf` 的 `%s` 动词能够格式化一个字符串。如果参数是一个 `error` 对象，则会自动调用其 `Error` 方法。

示例如下：

```go
fmt.Printf("%s", err)
```

但更好的方式是我们手动调用 `Error` 方法：

```go
fmt.Printf("%s", err.Error())
```

Python 之禅中的「显式胜于隐式」在这里依然适用。

##### 记录日志前请确保错误真的存在

如下示例代码记录了错误日志：

```go
package main

import (
    "fmt"
)

func Foo() error {
    return nil
}

func main() {
    err := Foo()
    // slog.Info(err.Error())
    fmt.Printf("INFO: call foo: %s\n", err)
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
call foo: %!s(<nil>)
```

可以发现这里格式化 `err` 对象失败了。

在将错误记录到日志前，我们有责任确保错误真的存在。否则调用 `err.Error()` 程序将发生 `panic`。

##### 何时记录错误日志

记录日志其实是一个比较大的话题，并且存在一定争议。何时记录、记录什么以及如何更好的记录都是比较复杂的问题。更糟糕的是，对于不同项目，可能会有不同的答案。

所以，这里只讲下我个人对何时记录错误日志的理解。

其实核心还是一句话：错误只应处理一次。

示例如下：

```go
func Foo() error {
    return errors.New("foo error")
}

func Bar() error {
    return Foo()
}

func main() {
    err := Bar()
    if err != nil {
        slog.Error(err.Error())
    }
}
```

示例代码中只在 `main` 函数中记录了一次错误日志，调用链中间遇到错误直接返回，不做任何处理。

同样，如果遇到服务降级的情况，我们也可以记录日志，并返回 `nil`，不再继续向上报告当前错误：

```go
func Foo() error {
    return errors.New("foo error")
}

func Bar() error {
    err := Foo()
    if err != nil {
        // NOTE: 服务降级，记录日志
        slog.Error(err.Error())
        return nil
    }
    // do something
    return nil
}

func main() {
    err := Bar()
    if err != nil {
        slog.Error(err.Error())
    }
}
```

此外，我们还可以在调用链中间为错误附加信息，并在顶层 `main` 函数中处理错误：

```go
func Foo() error {
    return errors.New("foo error")
}

func Bar() error {
    err := Foo()
    return errors.WithMessage(err, "Bar")
}

func main() {
    err := Bar()
    if err != nil {
        slog.Error(fmt.Sprintf("%+v", err))
    }
}
```

并且，这里还使用 `%+v` 动词记录了错误的详细堆栈信息到日志中。

##### 错误日志应该记录什么

一般来说，记录错误日志是为了方便排查问题，所以信息要尽可能多，那么错误日志中堆栈信息就必不可少。

所以，仍然推荐使用 `pkg/errors` 来处理错误。

如果项目比较小，调用层数不深，错误日志中只记录 `err.Error()` 信息也没什么关系。

但是，如果是中大型项目，尤其是微服务开发，错误日志应该记录 `err.Format()` 信息，即使用 `fmt.Sprintf("%+v", err)` 结果。

### pkg/errors

我在 [《Go 错误处理指北：pkg/errors 源码解读》](Go%20错误处理指北：pkg／errors%20源码解读.md) 一文中对 `pkg/errors` 包源码做了详细的讲解，不过却没有在用法上给出过多建议，本节就来补充一下。

pkg/errors 是由 Dave Cheney 所开发的，是目前 Go 错误处理的最优解。

#### 记录错误调用链

我们可以在错误调用链中，使用 `pkg/errors` 提供的 `errors.Wrap` 方法为错误附加一些信息，以此来记录链路调用过程。

示例如下：

```go
package main

import (
    "fmt"

    "github.com/pkg/errors"
)

func Foo() error {
    return errors.New("foo error")
}

func Bar() error {
    err := Foo()
    if err != nil {
        return errors.Wrap(err, "bar")
    }
    return nil
}

func main() {
    err := Bar()
    if err != nil {
        fmt.Printf("err: %s\n", err)
    }
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
err: bar: foo error
```

#### 记录错误堆栈

附加错误信息还不够，`pkg/errors` 包最大的好处是可以记录错误堆栈。

修改 `main` 函数的错误处理，只需要将 `fmt.Printf` 中格式化错误的动词从 `%s` 改成 `%+v` 即可：

```go
func main() {
    err := Bar()
    if err != nil {
        fmt.Printf("err: %+v\n", err)
    }
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
err: foo error
main.Foo
        /go/blog-go-example/error/handling-error/pkg-errors/main.go:32
main.Bar
        /go/blog-go-example/error/handling-error/pkg-errors/main.go:36
main.main
        /go/blog-go-example/error/handling-error/pkg-errors/main.go:44
runtime.main
        /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.1.darwin-arm64/src/runtime/proc.go:272
runtime.goexit
        /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.1.darwin-arm64/src/runtime/asm_arm64.s:1223
bar
main.Bar
        /go/blog-go-example/error/handling-error/pkg-errors/main.go:38
main.main
        /go/blog-go-example/error/handling-error/pkg-errors/main.go:44
runtime.main
        /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.1.darwin-arm64/src/runtime/proc.go:272
runtime.goexit
        /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.1.darwin-arm64/src/runtime/asm_arm64.s:1223
```

可以看到，错误从产生开始，整个调用链堆栈信息都被记录了下来。

但是这里存在重复的问题，错误调用链被打印了两次。这其实是因为 `pkg/errors` 包提供的 `errors.New` 函数本身在构造错误时就已经记录了堆栈信息，而 `errors.Wrap` 又记录了一遍。

所以，如果错误是通过 `errors.New` 构造的，调用链中间不应该再次使用 `errors.Wrap` 附加错误信息，而应该使用 `errors.WithMessage`。

修改 `Bar` 函数如下：

```go
func Bar() error {
    err := Foo()
    if err != nil {
        return errors.WithMessage(err, "bar")
    }
    return nil
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
err: foo error
main.Foo
        /go/blog-go-example/error/handling-error/pkg-errors/main.go:32
main.Bar
        /go/blog-go-example/error/handling-error/pkg-errors/main.go:36
main.main
        /go/blog-go-example/error/handling-error/pkg-errors/main.go:44
runtime.main
        /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.1.darwin-arm64/src/runtime/proc.go:272
runtime.goexit
        /go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.1.darwin-arm64/src/runtime/asm_arm64.s:1223
bar
```

现在记录的错误堆栈就正常了。

#### 不要做冗余的错误检查

其实 `pkg/errors` 包提供了更方便的使用方法。

我们无需编写这种代码：

```go
func Bar() error {
    err := Foo()
    if err != nil {
        return errors.WithMessage(err, "bar")
    }
    return nil
}
```

可以直接去掉那冗余的错误检查：

```go
func Bar() error {
    err := Foo()
    return errors.WithMessage(err, "bar")
}
```

这不对执行结果造成任何影响。

我们无需判断 `err` 是否为 `nil`，因为 `pkg/errors` 内部的方法帮我们做好了这项检查：

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
```

对于 `errors.Wrap/errors.WithStack` 同样如此。

#### Sentinel error 处理

因为 `pkg/errors` 包提供的 `errors.Wrap/errors.WithStack/errors.WithMessage` 这三个方法都会返回新的错误，所以默认情况下 `Sentinel error` 相等性判断就会失效。

不过 `pkg/errors` 包考虑到了这点，提供了 `errors.Cause` 方法可以得到一个错误的根因。

示例如下：

```go
func Foo() error {
    return io.EOF
}

func Bar() error {
    err := Foo()
    return errors.WithMessage(err, "bar")
}

func main() {
    err := Bar()
    if err != nil {
        if errors.Cause(err) == io.EOF {
            fmt.Println("EOF err")
            return
        }
        fmt.Printf("err: %+v\n", err)
    }
    return
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
EOF err
```

#### 小结

可以发现，`pkg/errors` 包充分考虑了人类和程序对错误的不同处理。

`pkg/errors` 包可以非常方便的向一个已有错误添加新的上下文，错误堆栈可以方便我们程序员排查问题，`errors.Cause` 获取错误根因的方法，可以方便程序中对错误进行相等性检查。

如果我们的代码中全局都在使用 `pkg/errors` 包，那么通过 `errors.New/errors.Errorf` 构造的错误天然就已经携带了错误堆栈信息。

通常在调用链中间过程直接返回底层错误即可，如果想要附加信息，则可以使用 `errors.WithMessage`，不要使用 `errors.Wrap/errors.WithStack` 以免造成堆栈信息的重复。

如果与标准库或来自第三方的代码包进行交互，可以考虑使用 `errors.Wrap/errors.WithStack` 在原错误基础上建立堆栈跟踪。

在错误处理调用链顶层，可以使用 `%+v` 来记录包含足够详细信息的错误。

### Go 1.13

Go 1.13 的发布为错误处理进带来了全新功能 `Error wrapping`。

具体细节可以参考相关提案 Proposal: Go 2 Error Inspection 以及 issues/29934 讨论。

Go 1.13 为 `errors` 和 `fmt` 标准库包引入了新功能：

- `fmt.Errorf` 支持使用 `%w` 动词包装错误。

- 新增 `errors.Unwrap` 函数为错误解包以获取根因。

- 新增 `errors.Is` 函数取代 `==` 做等值比较。

- 新增 `errors.As` 函数取代 `Type Assertion` 做类型断言。

我们来一一讲解。

#### fmt.Errorf

`fmt.Errorf` 新增的 `%w` 动词功能对标 `pkg/errors` 包中的 `errors.Wrap` 函数，用法如下：

```go
package main

import (
    "errors"
    "fmt"
)

func Foo() error {
    return errors.New("foo error")
}

func Bar() error {
    err := Foo()
    if err != nil {
        return fmt.Errorf("bar: %w", err)
    }
    return nil
}

func main() {
    err := Bar()
    if err != nil {
        fmt.Printf("err: %s\n", err)
    }
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
err: bar: foo error
```

#### errors.Unwrap

`errors.Unwrap` 函数对标 `pkg/errors` 包中的 `errors.Cause` 函数，用法如下：

```go
func Foo() error {
    return io.EOF
}

func Bar() error {
    err := Foo()
    if err != nil {
        return fmt.Errorf("bar: %w", err)
    }
    return nil
}

func main() {
    err := Bar()
    if err != nil {
        if errors.Unwrap(err) == io.EOF {
            fmt.Println("EOF err")
            return
        }
        fmt.Printf("err: %+v\n", err)
    }
    return
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
EOF err
```

#### errors.Is

`errors.Is` 函数可以取代双等号做等值比较，用法如下：

```go
func Foo() error {
    return io.EOF
}

func Bar() error {
    err := Foo()
    if err != nil {
        return fmt.Errorf("bar: %w", err)
    }
    return nil
}

func main() {
    err := Bar()
    if err != nil {
        // if err == io.EOF {
        if errors.Is(err, io.EOF) {
            fmt.Println("EOF err")
            return
        }
        fmt.Printf("err: %+v\n", err)
    }
    return
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
EOF err
```

由于我们使用了 `fmt.Errorf("bar: %w", err)` 对初始错误进行了包装，所以在 `main` 函数中不能直接使用 `if err == io.EOF` 来对 `Sentinel error` 进行相等性判断。

除了可以使用 `if errors.Unwrap(err) == io.EOF` 这种方式，我们还可以使用 `errors.Is(err, io.EOF)` 方式来取代双等号，执行结果相同。

#### errors.As

`errors.As` 函数可以取代 `Type Assertion` 做类型断言，用法如下：

```go
type MyError struct {
    msg string
    err error
}

func (e *MyError) Error() string {
    return e.msg + ": " + e.err.Error()
}

func Foo() error {
    return &MyError{
        msg: "foo",
        err: io.EOF,
    }
}

func Bar() error {
    err := Foo()
    if err != nil {
        return fmt.Errorf("bar: %w", err)
    }
    return nil
}

func main() {
    err := Bar()
    if err != nil {
        var e *MyError
        if errors.As(err, &e) {
            fmt.Printf("EOF err: %s\n", e)
            return
        }
        fmt.Printf("err: %+v\n", err)
    }
    return
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
EOF err: foo: EOF
```

### Go 1.20

Go 1.20 新增了 `errors.Join` 函数返回包装后的错误列表。

用法如下：

```go
package main

import (
    "errors"
    "fmt"
)

func main() {
    err1 := errors.New("err1")
    err2 := errors.New("err2")
    err := errors.Join(err1, err2)
    fmt.Println("---------")
    fmt.Println(err)
    fmt.Println("---------")
    if errors.Is(err, err1) {
        fmt.Println("err is err1")
    }
    if errors.Is(err, err2) {
        fmt.Println("err is err2")
    }
}
```

执行示例代码，得到输出如下：

```sh
$ go run main.go
---------
err1
err2
---------
err is err1
err is err2
```

可以发现通过 `errors.Join(err1, err2)` 得到的 `err` 对象打印结果中会在多个错误之间增加换行。并且 `errors.Is` 函数也做了升级，能够处理这种情况，对 `err1` 或 `err2` 的相等性比较结果都为 `true`。

### Go 1.23

Go 1.23 对 `errors.Is` 函数做了一点小优化。

这是 Go 1.23 中的 `errors.Is` 函数：

> https://github.com/golang/go/blob/go1.23.0/src/errors/wrap.go#L44-L51

```go
func Is(err, target error) bool {
    if err == nil || target == nil {
        return err == target
    }

    isComparable := reflectlite.TypeOf(target).Comparable()
    return is(err, target, isComparable)
}
```

这是 Go 1.22.8 中的 `errors.Is` 函数：

> https://github.com/golang/go/blob/go1.22.8/src/errors/wrap.go#L44-L51

```go
func Is(err, target error) bool {
    if target == nil {
        return err == target
    }

    isComparable := reflectlite.TypeOf(target).Comparable()
    return is(err, target, isComparable)
}
```

可以发现 Go 1.23 中 `errors.Is` 函数多了一个 `if err == nil` 逻辑的判断。不过由于这个改动比较小，不会对现有代码造成任何影响，这一改变并没有出现在 Go 1.23 Release Notes 中。

仅此而已。

### Go 2

Go 1.13 版本之前的错误处理是过去，Go 1.13 版本错误处理是现在，Go 2 版本错误处理是未来。

由于社区中对 Go 错误处理的吐槽声一直不断，Go 也在努力改变这一现状，虽然进展缓慢。

Go 1.13 的出现已经是最大的诚意了 :)。

很多人依然不满足于现状，将错误处理希望寄托于 Go 2。不过我个人持悲观态度，基于 Go 泛型的演进过程，我认为几年内 Go 错误处理都不会有较大改变。

不过 Go 2 的蓝图的确已经在勾勒中了，感兴趣的读者可以进入 Go 2 Draft Designs 查看，里面对 `Error handling` 以及 `Error values` 都罗列了几个链接供读者参阅。

期待 Go 2 的错误处理能够更上一层楼。

### 总结

Go 官方认为 `Error` 机制更简单有效，所以 Go 中并没有 `Exception`。

在 Go 中可以通过 `errors.New` 或 `fmt.Errorf` 构造一个错误对象。

有了错误对象，就需要对错误进行处理。Go 中一切的错误处理都从 `if err != nil` 开始。

`Sentinel error`，是一种错误处理惯用法，不过 `Sentinel error` 会成为公共 API 的一部分，它会引入源和运行时耦合，可能导致循环依赖。并且有些 `Sentinel error` 变量仅用于哨兵，而非错误，具有二义性，所以综合来看，并不推荐使用。尽管标准库中有大量应用，但是为了保持 Go 1 的兼容性承诺，短期来看不太可能有大变动。

因为 `Sentinel error` 是一个变量，值可以被改变。所以有人提出使用常量来定义错误，不过目前来看这种用法并不常见。

`Opaque error` 是最理想的错误处理方式，但它过于理想。

为了提供更多的上下文信息，我们可以自定义错误类型。

始终记住：`Errors are values`，无论是内置错误类型还是我们自定义的错误类型都是值，我们可以对其进行等值判断、类型断言、赋值给结构体的属性等操作。

随着 Go 语言的演进，现在社区中更推荐断言错误行为。这样能够实现了我们的代码与错误所在的包之间最大化的解耦。

在使用 `Builder 模式`、链式调用或者 `for` 循环等场景下，暂存中间过程所出现的错误，有助于简化代码，使编写出的代码逻辑更加连贯。

为了避免函数返回值存在二义性，我们应该返回错误而不是指针。

不要忽略你的错误，但错误也不应该被重复处理，错误只应被处理一次。

在在缩进代码块中处理错误有助于正常逻辑清晰可见。

由于 Go 接口实现的特殊方式，可能存在 `nil` 错误值可能不等于 `nil` 的情况，编写代码时需要注意不要踩坑。

记录日志前请确保错误真的存在，一个错误只应记录一次日志，并且为了方便拍错，日志信息最好包含堆栈信息。

Go 1.13 虽然新增了几个方法，用来辅助错误处理，但是依然不能记录堆栈信息。

所以，即使有 Go 1.13 的存在，现在也依然推荐使用 `pkg/errors` 来处理错误。

Go 2 承担了 Go 未来错误处理的重任。

本文示例源码我都放在了 GitHub 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- EOF 是什么？：https://www.ruanyifeng.com/blog/2011/11/eof.html
- Why does Go not have exceptions?：https://go.dev/doc/faq#exceptions
- Type Assertion：https://go.dev/doc/effective_go#interface_conversions
- Type switch：https://go.dev/doc/effective_go#type_switch
- Why is my nil error value not equal to nil?：https://go.dev/doc/faq#nil_error
- Return and handle an error：https://go.dev/doc/tutorial/handle-errors
- Error wrapping：https://go.dev/doc/go1.13#error_wrapping
- Error handling in Upspin：https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html
- Why Go's Error Handling is Awesome：https://rauljordan.com/why-go-error-handling-is-awesome/
- Error Handling in Go：https://medium.com/gett-engineering/error-handling-in-go-53b8a7112d04
- Handle Errors In Go Like A Pro：https://medium.com/@leodahal4/handle-errors-in-go-like-a-pro-5f2ab97c660b
- The Go standard error APIs：https://dr-knz.net/cockroachdb-errors-std-api.html
- Constant errors：https://dave.cheney.net/2016/04/07/constant-errors
- Inspecting errors：https://dave.cheney.net/2014/12/24/inspecting-errors
- Errors and Exceptions, redux：https://dave.cheney.net/2015/01/26/errors-and-exceptions-redux
- Gocon Spring 2016：https://dave.cheney.net/paste/gocon-spring-2016.pdf
- Stack traces and the errors package：https://dave.cheney.net/2016/06/12/stack-traces-and-the-errors-package
- Don’t just check errors, handle them gracefully：https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully
- Effective error handling in Go.：https://morsmachine.dk/error-handling
- Design Philosophy On Logging：https://www.ardanlabs.com/blog/2017/05/design-philosophy-on-logging.html
- Error Handling In Go, Part I：https://www.ardanlabs.com/blog/2014/10/error-handling-in-go-part-i.html
- Error Handling In Go, Part II：https://www.ardanlabs.com/blog/2014/11/error-handling-in-go-part-ii.html
- Design Philosophy On Logging：https://www.ardanlabs.com/blog/2017/05/design-philosophy-on-logging.html
- Error handling and Go：https://go.dev/blog/error-handling-and-go
- Errors are values：https://go.dev/blog/errors-are-values
- Working with Errors in Go 1.13：https://go.dev/blog/go1.13-errors
- Defer, Panic, and Recover：https://go.dev/blog/defer-panic-and-recover
- Go 1.13 errors 源码：https://github.com/golang/go/tree/go1.13/src/errors
- Go 1.22.8 errors 源码：https://github.com/golang/go/blob/go1.22.8/src/errors/wrap.go#L44-L51
- Go 1.23 errors 源码：https://github.com/golang/go/blob/go1.23.0/src/errors/wrap.go#L44-L51
- Go 错误处理指北：Error vs Exception vs ErrNo：https://jianghushinian.cn/2024/09/06/go-error-guidelines-error-exception-errno/
- Go 错误处理指北：pkg/errors 源码解读：https://jianghushinian.cn/2024/09/14/go-error-guidelines-pkg-errors/
- 如何规范 RESTful API 的业务错误处理：https://jianghushinian.cn/2023/03/04/how-to-standardize-the-handling-of-restful-api-business-errors/
- Builder 模式在 Go 语言中的应用：https://jianghushinian.cn/2024/08/26/go-design-patterns-builder/
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/error/handling-error

**联系我**

- 公众号：Go 编程世界
- 微信：jianghushinian
- 邮箱：jianghushinian007@outlook.com
- 博客：https://jianghushinian.cn
