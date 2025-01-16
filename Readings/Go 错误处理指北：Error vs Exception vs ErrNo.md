---
title: Go 错误处理指北：Error vs Exception vs ErrNo
author:
  - Go 编程世界
date created: 2025-01-01 14:37:01
category: Golang
tags:
  - "#错误处理"
  - "#编程实践"
  - "#Golang"
  - "#Python"
url:
  - https://mp.weixin.qq.com/s/d6Fcksd73SpcZMQy4tc-sA
description: 这篇文章深入探讨了Go、Python和C语言中的错误处理机制，通过对比分析，帮助读者理解每种语言在错误处理上的设计哲学和实际应用。文章不仅展示了各语言的典型错误处理代码示例，还讨论了各自的优缺点，并提供了一些实用的编程建议。无论是初学者还是有经验的开发者，都能从中获得对错误处理更深刻的理解和启发。
status: Finished
---

很多有其他编程语言经验的人初次接触 Go 语言时，想必对 `if err != nil` 的错误处理方式感到新奇，之后用久了，竟发现有点令人抓狂。

因为很多人不满 Go 语言的错误处理方式，甚至有人做了一张梗图：

![](../Assets/Images/GoErrorHandlingGuide_p1.png)

哈哈😄，不吹不黑，本文就来对比下 Python、C 以及 Go 这三种编程语言中的异常处理机制，看看你更喜欢哪一种。

### Python 错误处理

因为我接触的第一门编程语言是 Python，所以我就先讲讲 Python 中的错误处理机制。

Python 的错误处理机制与 Java、C#、JavaScript 等主流的高级编程语言非常类似，它们都可以算做是 Exception 派系。

以下是 Python 错误处理的典型示例程序：

```python
def div(a, b):
    return a / b

try:
    result = div(1, 0)
    print(result)
except ZeroDivisionError as e:
    logging.error(e)
except Exception as e:
    logging.error(e)
```

`div` 函数内部不对参数进行任何校验，当除数 `b` 为 `0` 时，代码会抛出 `ZeroDivisionError` 错误。

我们在函数调用处使用 `try…except…` 语句对错误进行捕获并处理。捕获到 `ZeroDivisionError` 表示遇到除数为 `0` 的情况，而捕获到 `Exception` 则为了避免放过出现其他未知的异常。

从这两个错误的命名来看：`ZeroDivisionError` 和 `Exception`，一个表示「错误」，一个表示「异常」。其实它们都继承自 `BaseException` 基类，在 Python 中并不区分错误和异常，所以 Python 中的错误处理，我们一般称为异常处理。所有 Exception 派系的编程语言也都类似。

除了内置异常，我们也可以很方便的定义自己的异常类：

```python
class MyException(Exception):
    ...
```

没错，就是这么简单。

可以按照如下方式使用自定义异常：

```python
raise MyException("my custom exception")
```

这与内置异常没什么两样，`try…except…` 同样能够正常捕获自定义异常。

其实这种 `try…except…` 方式处理异常，最大的好处就是：**异常兜底**。

在一整个代码块中间，无论写了多少行代码，无论哪里可能出现异常，我们都可以放心大胆的不去处理异常，而是在代码块最外层使用 `try…except…` 进行捕获，示例如下：

```python
def a():
    ...

def b():
    ...

def c():
    ...

def main():
    try:
        a()
        b()
        c()
    except Exception as e:
        logging.error(e)
    else:
        print("success")
    finally:
        print("release resources")
```

这种方式极大的简化了我们的代码编写，不用在每一个函数调用处都对异常进行处理，仅需要在最外层做一次异常处理即可。

当然，这种方式也有弊端，那就是异常不再是异常。什么意思？就是说，在 Python 中的异常其实是不分轻重的，有些异常可以忽略，有些异常又不可以忽略，但上面的示例代码显然无法区分异常的严重程度。

并且，因为有异常兜底，很多开发者写代码的时候就会更加随性，写出来的程序就会更加不可控。

接下来，我们再看一看 C 语言中的错误处理。

### C 错误处理

虽然我在工作中没怎么写过 C，但 C 乃 “万物鼻祖”。既然要对比多种编程语言错误处理机制，C 自然必不可少。

其实 C 语言并没有提供对错误处理的直接支持。一般来说，在 C 语言内部函数代码出现错误时，会返回 `1` 或 `NULL`，并且同时会设置一个错误码 `errno`，以此来表示错误类型。

相比于其他高级语言对错误的抽象，C 语言的错误处理就显得比较 “原始” 了。

以下是 C 语言错误处理的典型示例程序：

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>  // 包含 strerror 函数的头文件

int main() {
    // 清除 errno 初始值，这是一个好的编程习惯
    errno = 0;

    FILE *file;
    char *filename = "example.txt";

    // 尝试以读取模式打开文件
    file = fopen(filename, "r");
    if (file == NULL) {
        // 打开文件出错
        printf("Failed to open file: %s\n", filename);
        // 查看错误码 errno
        printf("ErrNo: %d\n", errno);
        // perror 函数显示传入的字符串，后跟一个冒号、一个空格和当前 errno 值的文本表示形式
        perror("Error");
        // strerror 函数返回一个指针，指向当前 errno 值的文本表示形式
        printf("Error: %s\n", strerror(errno));
        // 在发生错误时，大多数的 C 或 UNIX 函数调用返回 1 或 NULL
        return 1;
    }

    // 文件操作
    printf("open file success\n");

    // process file...

    // 完成操作后，关闭文件
    fclose(file);

    return 0;
}
```

我们尝试使用 `fopen` 来打开一个文件，`fopen` 函数返回值 `file` 可能是 `FILE*` 表示文件描述符，也可能是 `NULL` 表示函数出错。

所以我们需要通过 `if (file == NULL)` 来判断打开文件是否出错，如果出错，全局变量 `errno` 会被赋值，使用 `perror` 函数或者 `strerror(errno)` 可以读取错误码对应的错误描述信息。

> NOTE:`perror()` 函数打印传入的字符串，后跟一个冒号、一个空格和当前 `errno` 值的文本表示形式。`strerror()` 函数，返回一个指针，指针指向当前 `errno` 值的文本表示形式。

NOTE: 我们能在 `<errno.h>` 头文件中找到错误码定义：

```c
/*
 * Error codes
 */

#define EPERM           1               /* Operation not permitted */
#define ENOENT          2               /* No such file or directory */
#define ESRCH           3               /* No such process */
#define EINTR           4               /* Interrupted system call */
#define EIO             5               /* Input/output error */
#define ENXIO           6               /* Device not configured */
#define E2BIG           7               /* Argument list too long */
...
```

执行示例代码，打开一个不存在的文件，输出如下：

```sh
$ gcc main.c -g -o main
$ ./main
Failed to open file: example.txt
ErrNo: 2
Error: No such file or directory
Error: No such file or directory
```

这种错误处理方式存在一个很大的问题：**返回值有二义性**。

`fopen` 函数即可能返回 `FILE*` 也可能返回 `NULL`，这是一种很糟糕的做法。

并且，很多人会因此忘记错误检查。

而在 C 语言中，之所以要这样处理错误，主要是因为 C 语言的函数只能有一个返回值。所以这也是一种没有办法的办法。

不过，这种方式并不能处理所有错误场景。

如果用 C 来实现 `div` 函数，就不能使用 `NULL` 作为返回值，`NULL` 无法转换成 `int` 类型。

我们只能这样做：

```c
#include <stdio.h>

int div(int a, int b, int *result) {
    if (b == 0) {
        return -1; // 返回 -1 表示错误
    }
    *result = a / b; // 将结果存储在指针所指向的变量中
    return 0; // 返回 0 表示成功
}

int main() {
    int result;
    int err;

    err = div(1, 0, &result);

    // 错误处理
    if (err == -1) {
        printf("division by zero\n");
    } else {
        printf("result: %d\n", result);
    }

    return 0;
}
```

`div` 函数还是单返回值，不过这个返回值只代表错误，返回 `0` 表示成功，返回 `-1` 表示失败。

> 要我说其实这就很 Go 了

`div` 函数计算结果通过参数的形式，被存储在指针所指向的变量 `int *result` 中。

这也是 C 语言中错误处理的另一种典型做法。

但这种用参数接收计算结果的方式，我个人其实是不太喜欢的，因为我认为语义不够清晰。

> NOTE: 其实 C 函数支持返回结构体指针，这样虽然函数只能返回一个值，但可玩性就大大增加了，所以单返回值也不是不能接受。

最后，我们再看一看 C 语言中的错误处理。

### Go 错误处理

我在前文说 C 是 “万物鼻祖”，这在编程世界里并不算太夸张的说法。Python 解释器就是用 C 语言写的，而 Go 语言作者之一 肯・汤普逊 同时又是 Unix 操作系统和 B 语言（C 语言的前身）作者，所以 Go 在设计之初，就大量参考了 C 语言的设计。

不难推测，Go 的错误处理也参考了 C 语言。事实也的确如此，Go 的错误处理继承自 C 语言，并在其基础上做了改进。

以下是 Go 语言错误处理的典型示例程序：

```go
package main

import (
    "errors"
    "fmt"
    "log"
)

func div(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

func main() {
    result, err := div(1, 0)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(result)
}
```

得益于 Go 函数支持多返回值，所以 Go 就不再需要 `errno` 了。

Go 定义了一个 `error` 类型，专门用来表示错误，使用 `errors.New` 可以轻松构建一个错误对象。

这也就引出了经典的 `if err != nil`，程序遇到错误就会走入此分支，我们需要在此做错误处理。

Go 中的 `error` 其实是一个接口：

```go
type error interface {
    Error() string
}
```

任何实现了 `error` 接口的类型，都可以是一个错误。

而我们通过 `errors.New` 创建的错误，其实就是一个内置错误类型的实现：

```go
func New(text string) error {
    return &errorString{text}
}

type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

我们也完全可以根据自己的需要，自定义 `error`：

```go
type MyError struct {
    msg string
}

func (e *MyError) Error() string {
    return e.msg
}
```

并且 `MyError` 中可以增加任何我们想要的字段和方法。

我们可以像这样处理不同类型的错误：

```go
if err != nil {
    switch err.(type) {
    case *MyError:
        // ...
    case *ZeroDivisionError:
        // ...
    default:
        // ...
    }
}
```

可以发现，Go 中的错误处理其实是对返回值的检查，并且我们可以通过类型断言 `err.(type)` 来判断 `error` 的具体类型。

有时候，我们想忽略错误，需要这样做：

```go
result, _ := div(1, 0)
```

在 Go 中必须使用 `_` 显式的忽略错误，否则如果不处理 `err` 程序会编译不通过。

因为 Go 函数可以返回多个值，所以这给我们处理错误带来了更大的灵活性，比如下面这个示例：

```go
type SmsCode struct{}

func (c SmsCode) Verify1() (bool, error) {
    // ...
}

func (c SmsCode) Verify2() error {
    // ...
}
```

假如我们有一个发送短信验证码的结构体 `SmsCode`，需要为其定义 `Verify` 方法来验证用户输入的短信验证码是否正确。

我们可以定义成 `Verify1` 这样，方法返回两个值，`bool` 值表示验证码是否正确，`error` 值表示代码执行过程中是否出错，比如 Redis 无法调通等。

我们也可以定义成 `Verify2` 这样，方法仅返回一个值，验证码错误也可以当作是一种 `error` 类型，比如 `errors.New("invalid sms code")`。

当然，这就是业务上需要我们自己做决策的地方了，和语言本身错误处理无关。这也正是因为 Go 提供的方便，让我们比在写 C 代码时可以更加灵活的处理错误。

不过，Go 中的错误处理也有弊端，看下面这个示例你就有体会了：

```go
func a() error {
    // ...
}

func b() error {
    // ...
}

func c() error {
    // ...
}

func main() {
    err := a()
    if err != nil {
        log.Fatal(err)
    }
    err = b()
    if err != nil {
        log.Fatal(err)
    }
    err = c()
    if err != nil {
        log.Fatal(err)
    }
}
```

是不是有点抓狂。

哈哈，其实只要我们做好封装，还是可以避免这种代码产生的。

相较于 Python，Go 对错误处理更加谨慎，我们不能通过在代码块最外层写一个 `try…except…` 来进行兜底，只能一步一步去处理错误。

此外，Python 在异常处理中提供了 `finally` 语句可以方便我们释放资源，不论代码是否执行出错，`finally` 语句最终都会执行。

Go 也提供了类似功能，即 `defer` 语句：

```go
func main() {
    defer func() {
        fmt.Println("release resources")
    }()

    result, err := div(1, 0)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(result)
}
```

执行示例代码：

```sh
$ go run main.go
division by zero
release resources
```

被 `defer` 修饰的函数会在外层的 `main` 函数退出前被调用执行。

> NOTE: C 语言可以使用 `goto fail;` 跳转到指定代码块来进行资源释放。

另外，Go 与其他主流编程语言在错误处理上有一个很大的分歧，Go 区分了「错误」和「异常」。

```go
package main

import "fmt"

func mayPanic() {
    panic("a problem")
}

func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered error:", r)
        } else {
            fmt.Println("recovered success")
        }
    }()

    mayPanic()

    fmt.Println("after mayPanic()")
}
```

在 Go 中 `panic` 表示一个异常，与 `error` 错误不同，默认情况下，遇到 `panic` 调用，Go 程序会崩溃并退出。

可以使用 `recover` 来捕获 `panic` 抛出的异常，但是要注意 `recover` 一定要放在函数入口处的 `defer` 语句中，因为只有这样才能保证 `recover` 会被调用。

我们可以在主动捕获异常的地方，自行处理出现异常后的逻辑。

执行示例代码：

```sh
$ go run main.go  
recovered error: a problem
```

可以发现 `after mayPanic()` 并没有被打印。这说明代码执行到 `panic` 以后就中断了，`panic` 会抛出异常，然后异常被 `recover` 捕获并处理。

`recover` 可以看作是一种全局的兜底异常处理方案，Gin 框架就通过中间件的方式，在处理请求的时候 `recover` 住未知的异常，以防程序退出。

不过绝大多数情况下并不建议使用 `panic` + `recover` 机制，`panic` 表示意外的异常，而我们在编写代码阶段，更多需要处理的情况是可以预见的错误 `error`。

对于真正意外的情况，比如数组索引越界，我们才会使用 `panic`，对于一般性错误，我们应该是使用 `error` 来进行处理。

以上便是 Go 语言的错误处理机制。

### 总结

Python 的错误处理方式在主流编程语言中最为常见，只不过其他编程语言的关键字一般为 `try…catch…finally`。

这种异常处理方式的好处是代码简洁优雅，所以是最被程序员所接受的一种错误处理方式。

不知道你有没有注意，我在 Python 异常处理的示例代码中，还写了一个 `else` 分支，其实 Python 异常处理完整逻辑是 `try…except…else…finally`，`else` 分支的功能你可以自己思考下是干什么用的。

C 的错误处理比较 “原始”，很大原因是 C 函数只能返回单个值。所以就可能出现一个返回值，存在多种用途的现象。不过既然都用 C 语言开发程序了，这一点显然是要程序员自己要解决的问题。

Go 因为是后起之秀，所以可以参考它的前辈们，来设计属于自己风格的错误处理机制。不过即使这样，Go 的错误处理仍然是被吐槽最多的。

我倒是觉得这种错误处理非常的“Go”，很有 Go 语言的特点，大道至简。

对比了三种主流编程语言的错误处理以后，你是否对 Go 语言的错误处理有了更新的认识？你喜欢哪种错误处理方式？可以在评论区进行交流。

本文示例源码我都放在了 GitHub 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- The Python Tutorial：Errors and Exceptions：https://docs.python.org/3/tutorial/errors.html

- C 错误处理：https://www.runoob.com/cprogramming/c-error-handling.html

- Go 编程模式：错误处理：https://coolshell.cn/articles/21140.html

- The Go Blog：Defer, Panic, and Recover：https://go.dev/blog/defer-panic-and-recover

- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/error/error-exception-errno

**联系我**

- 公众号：Go 编程世界
- 微信：jianghushinian
- 邮箱：jianghushinian007@outlook.com
- 博客：https://jianghushinian.cn
