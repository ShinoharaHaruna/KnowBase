---
title: Golang 中的 Context 如何让代码更智能、更安全、更易于扩展
author: 南丞
date created: 2024-12-28T17:51:00
category: Golang
tags:
  - "#Golang"
  - "#编程实践"
  - "#并发"
  - "#软件开发"
  - "#代码优化"
url:
  - https://mp.weixin.qq.com/s/uW7NW-BMk32fDxLiM8PByQ
description: 这篇文章详细介绍了在 Golang 中如何使用 context 包来增强代码的智能性、安全性和可扩展性。通过探讨 context 的基本功能、使用方法以及高级应用，读者可以学习如何在 Go 语言中有效地管理 goroutine 之间的交互，防止资源泄漏，并简化函数参数传递。
status: 已读完
---

## Golang 中的 Context 如何让代码更智能、更安全、更易于扩展

Go 开发中，context 包已经成为一个必不可少的工具。它提供了==一种在不同的 goroutine 之间传递请求范围内变量、取消信号和截止时间的方法==。通过合理地使用 context，我们可以使代码变得更智能、更安全，并且更易于扩展。本文将详细探讨 context 的作用以及如何在实际开发中应用它。

## 什么是 Context？

context 是 Go 1.7 引入的一个标准库，它主要用于在 goroutine 之间传递请求范围内的变量以及控制信号。context 主要有以下几个功能：

- 取消信号传递：可以通过 context 传递取消信号，用于取消正在进行的操作。
- 截止时间传递：可以设定一个截止时间，超时后会自动取消操作。
- 请求范围内变量传递：可以在 context 中传递一些与请求相关的变量。

## context 的基本使用

### 创建 Context

context 包提供了四种创建 context 的方法：

- `context.Background()`：返回一个空的 `Context`，一般用于主函数、初始化和测试。
- `context.TODO()`：返回一个空的 `Context`，表示目前还不知道用什么 Context 时使用。
- `context.WithCancel(parent)`：返回一个可取消的 context 和一个取消函数 cancel。
- `context.WithDeadline(parent, deadline)` 和 `context.WithTimeout(parent, timeout)`：返回一个带有超时功能的 context。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// 创建一个带有取消功能的 context
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // 确保在函数结束时取消 context

	go func() {
		time.Sleep(2 * time.Second)
		cancel() // 2 秒后取消 context
	}()

	select {
	case <-time.After(3 * time.Second):
		fmt.Println("Operation completed")
	case <-ctx.Done():
		fmt.Println("Operation cancelled")
	}
}
```

在上述代码中，创建了一个带有取消功能的 context。在另一个 goroutine 中，在 2 秒后取消了 context，所以 select 语句会在 ctx.Done () 信道接收到取消信号时执行相应的操作。

### 如何使用 Context 使代码更智能

通过使用 context，可以在不同的 goroutine 之间传递控制信号和变量，这样可以减少全局变量的使用，使代码更加模块化和智能。例如，在处理 HTTP 请求时，可以将请求的上下文传递给所有处理函数，从而确保在请求取消时，所有相关的操作都能及时地响应并终止。

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	fmt.Println("Handler started")
	defer fmt.Println("Handler ended")

	select {
	case <-time.After(5 * time.Second):
		fmt.Fprintf(w, "Hello, World!")
	case <-ctx.Done():
		err := ctx.Err()
		fmt.Println("Handler cancelled:", err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}

func main() {
	http.HandleFunc("/", handler)
	http.ListenAndServe(":8080", nil)
}
```

在上述代码中，将 HTTP 请求的 context 传递给处理函数 handler，这样当请求被取消时，处理函数可以及时响应并终止操作。

### 如何使用 Context 使代码更安全

context 提供的取消和超时机制，可以有效地防止资源泄漏和僵尸进程。例如，在进行数据库查询或网络请求时，如果操作超时或请求被取消，可以及时终止操作，释放资源。

> 这里原文给的例子，我稍微简化了一下

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func queryWithTimeout(ctx context.Context) error {
	ctx, cancel := context.WithTimeout(ctx, 1*time.Second) // 用 main 中的空 context 作为基础产生带超时的 context
	defer cancel()

	// 模拟数据库查询，用 sleep 代替
	select {
	case <-time.After(2 * time.Second):
		fmt.Println("查询成功完成")
		return nil
	case <-ctx.Done():
		return ctx.Err() // 默认的 err 是 context deadline exceeded
	}
}

func main() {
	ctx := context.Background() // 创建一个空的 Context
	err := queryWithTimeout(ctx)
	if err != nil {
		fmt.Println("查询失败:", err)
		return
	}
}
```

在上述代码中，在数据库查询时使用了 `context.WithTimeout`，这样如果查询时间超过 1 秒，查询操作会自动取消并返回超时错误，从而避免了长时间阻塞。

### 如何使用 Context 使代码更易于扩展

通过使用 context，可以方便地在不同的函数之间传递信息，而不需要修改函数签名。这使得代码更易于扩展和维护。例如，可以在 context 中传递一些用户认证信息或请求 ID，从而简化函数参数。

```go
package main

import (
	"context"
	"fmt"
)

type key int

const requestIDKey key = 0

func withRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func requestIDFromContext(ctx context.Context) (string, bool) {
	// 上下文对象可以视作一个键值对的集合，通过键值对的方式存储和获取数据
	requestID, ok := ctx.Value(requestIDKey).(string)
	return requestID, ok
}

func handleRequest(ctx context.Context) {
	if requestID, ok := requestIDFromContext(ctx); ok {
		fmt.Println("Handling request with ID:", requestID)
	} else {
		fmt.Println("No request ID found in context")
	}
}

func main() {
	ctx := context.Background()
	ctx = withRequestID(ctx, "12345")
	handleRequest(ctx)
}
```

在上述代码中，使用 `context.WithValue` 在 context 中存储了一个请求 ID，然后在处理函数中提取并使用这个请求 ID。这使得可以在不修改函数签名的情况下，方便地传递和使用请求范围内的变量。

> 说起来，一开始改写这个代码的时候，对于上下文够改用了指针。然而实际上，涉及 `context.Context` 的函数使用指针 `*context.Context`，这是不必要的，也是不正确的。
>
> 在 Go 中，`context.Context` 是一个接口类型，按值传递是惯例。
>
> 1. **接口本质**：接口本身已经是一个指针，指向具体实现。因此，再使用指针没有意义，会导致不必要的复杂性。
> 2. **不可变性**：`context.Context` 设计为不可变的。通过按值传递，确保上下文在传递过程中不会被意外修改。
> 3. **惯用法**：Go 语言的惯用法是按值传递 `context.Context`，这使得代码更简洁且符合社区标准。

## Context 的高级使用

### 传递元数据

有时需要在多个 goroutine 之间传递元数据（如请求 ID、用户认证信息等）。context 可以用来安全地传递这些信息。

例子同上。使用 `context.WithValue` 在 context 中存储了一个请求 ID，然后在处理函数中提取并使用这个请求 ID。这使得我们可以在不修改函数签名的情况下，方便地传递和使用请求范围内的变量。

### 处理并发操作

在处理并发操作时，context 可以帮助控制 goroutine 的生命周期，确保在请求取消时能够正确地终止所有相关操作。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func worker(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("%s: received cancellation signal\n", name)
			return
		default:
			fmt.Printf("%s: working...\n", name)
			time.Sleep(1 * time.Second)
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	go worker(ctx, "worker1")
	go worker(ctx, "worker2")

	time.Sleep(3 * time.Second)
	fmt.Println("Cancelling context")
	cancel()

	// 等待一段时间以确保所有 goroutine 都能收到取消信号
	time.Sleep(1 * time.Second)
}
```

在上述代码中，创建了两个并发执行的 worker goroutine，并使用 `context.WithCancel` 创建了一个可取消的 context。当主函数调用 cancel () 时，所有的 worker 都会接收到取消信号并停止工作。

线程 `worker` 在一个无限循环中，使用 `select` 语句监听上下文的取消信号。如果收到取消信号 (`ctx.Done()`)，打印取消信息并返回，终止该 goroutine。

调用 `cancel()` 函数会触发以下过程：

1. **关闭 `Done` 通道**：`ctx.Done()` 返回的通道会被关闭。任何正在监听这个通道的 goroutine 都会收到通知。
2. **通知 goroutines**：所有监听这个上下文的 goroutine 会检测到 `Done` 通道已关闭，从而执行相应的停止操作。
3. **释放资源**：取消上下文后，相关资源可以被释放，防止内存泄漏。

调用 `cancel()` 会关闭 `ctx.Done()` 返回的通道，从而使得 `case <-ctx.Done()` 可以收到通知并执行相应的处理逻辑。这是 `context` 的一种机制，用于优雅地取消和停止 goroutine 的执行。

### 使用 Context 进行超时控制

在处理外部资源（如网络请求、数据库查询）时，设置超时是非常重要的。使用 context 可以轻松地实现超时控制。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func fetchURL(ctx context.Context, url string) error {
	select {
	case <-time.After(2 * time.Second): // 模拟耗时操作
		fmt.Printf("Fetched %s: %s\n", url, "200 OK")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	url := "https://www.google.com"
	if err := fetchURL(ctx, url); err != nil {
		fmt.Println("Error fetching URL:", err)
	}
}
```

在上述代码中，使用 `context.WithTimeout` 创建了一个带有超时功能的 context，并将其传递给 fetchURL 函数。如果请求超过了 1 秒，context 将自动取消请求，避免了长时间的阻塞。

## Context 的最佳实践

在使用 Context 时，有一些重要的最佳实践需要遵循：

**不要将 Context 存储在结构体中**

```go
// 错误示例
type Service struct {
    ctx context.Context    // 不要这样做
}

// 正确示例
type Service struct {
    // ... 其他字段
}

func (s *Service) DoSomething(ctx context.Context) error {
    // 在方法参数中传递 context
}
```

**Context 应该是函数的第一个参数**

```go
// 推荐的方式
func DoSomething(ctx context.Context, arg string) error {
    // ...
}

// 不推荐的方式
func DoSomething(arg string, ctx context.Context) error {
    // ...
}
```

**使用 context.WithValue 时要谨慎**

```go
// 推荐：使用自定义类型作为 key  
type contextKey string  
const userIDKey contextKey = "userID"  
  
// 不推荐：直接使用内置类型作为 key  
ctx = context.WithValue(ctx, "userID", "123") // 避免这样做
```

## 常见陷阱和注意事项

1. **避免传递 nil context**：总是使用 `context.Background()` 或 `context.TODO()` 作为起点
2. **注意 context 取消的传播**：父 context 取消时，所有子 context 都会被取消
3. **合理使用超时设置**：避免设置过长或过短的超时时间
4. 正确处理 `context.Done()`：在使用 select 语句时，确保正确处理取消信号

### 总结

通过合理地使用 context，可以使 Go 代码变得更智能、更安全，并且更易于扩展。 context 提供的取消信号、超时机制和变量传递功能，使得可以更好地控制并管理 goroutine 之间的交互，从而编写出更加健壮和可靠的程序。在实际开发中，建议尽量使用 context 处理与请求范围相关的操作，以提高代码的可维护性和扩展性。
