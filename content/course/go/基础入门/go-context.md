https://github.com/cch123/golang-notes/blob/master/context.md

# Go Context 源码分析

## 概述

### Context是干什么的

- context 用来在 groutine 之间传递上下文信息。
- context 几乎成为了并发控制和超时控制的标准做法。
- context 可以协调多个 groutine 中的代码执行，并且可以存储键值对。
- context 是并发安全的。

### 为什么有 Context



### 官方介绍

Context 包定义了一个 Context 类型，它携带了deadline、cacellation信号和其他的跨API边界、不同进程请求域内的值。

进入服务端的请求应该携带一个 Context，被调用的服务端接受一个 Context 参数，在这条调用链上传播这个 Context，或者选择性的被一个用 WithCancel、WithDeadline、WithTimeout创建的派生Context代替。当Context被取消时，所有派生的Context也将被取消。

WithCancel、WithDeadline、WithTimeout 函数会基于父Context创建一个子Context和一个CancelFunc函数。

开发者在使用Context时应该遵循以下原则以保持接口的一致性，并且使静态分析工具检查Context的传播：

- 不要在结构体外存储 Context，而是应该在每一个需要它的函数中显示的传递Context，并且Context应该作为第一个参数传递，形如：func SomeThing(cxt context.Context, arg Arg)
- 不要传递一个 nil 的Context，即使函数允许为nil。如果你不确定用什么Context，传递一个Context.TODO

同一个Context可以在不同的groutine之间传递，Context在不同的协程之间同时使用是安全的

## 源码分析

源码位置：src/context/context.go

```go
type Context interface {
	// Deadline returns the time when work done on behalf of this context
	// should be canceled. Deadline returns ok==false when no deadline is
	// set. Successive calls to Deadline return the same results.
	Deadline() (deadline time.Time, ok bool)

	// Done returns a channel that's closed when work done on behalf of this
	// context should be canceled. Done may return nil if this context can
	// never be canceled. Successive calls to Done return the same value.
	// The close of the Done channel may happen asynchronously,
	// after the cancel function returns.
	//
	// WithCancel arranges for Done to be closed when cancel is called;
	// WithDeadline arranges for Done to be closed when the deadline
	// expires; WithTimeout arranges for Done to be closed when the timeout
	// elapses.
	//
	// Done is provided for use in select statements:
	//
	//  // Stream generates values with DoSomething and sends them to out
	//  // until DoSomething returns an error or ctx.Done is closed.
	//  func Stream(ctx context.Context, out chan<- Value) error {
	//  	for {
	//  		v, err := DoSomething(ctx)
	//  		if err != nil {
	//  			return err
	//  		}
	//  		select {
	//  		case <-ctx.Done():
	//  			return ctx.Err()
	//  		case out <- v:
	//  		}
	//  	}
	//  }
	//
	// See https://blog.golang.org/pipelines for more examples of how to use
	// a Done channel for cancellation.
	Done() <-chan struct{}

	// If Done is not yet closed, Err returns nil.
	// If Done is closed, Err returns a non-nil error explaining why:
	// Canceled if the context was canceled
	// or DeadlineExceeded if the context's deadline passed.
	// After Err returns a non-nil error, successive calls to Err return the same error.
	Err() error

	// Value returns the value associated with this context for key, or nil
	// if no value is associated with key. Successive calls to Value with
	// the same key returns the same result.
	//
	// Use context values only for request-scoped data that transits
	// processes and API boundaries, not for passing optional parameters to
	// functions.
	//
	// A key identifies a specific value in a Context. Functions that wish
	// to store values in Context typically allocate a key in a global
	// variable then use that key as the argument to context.WithValue and
	// Context.Value. A key can be any type that supports equality;
	// packages should define keys as an unexported type to avoid
	// collisions.
	//
	// Packages that define a Context key should provide type-safe accessors
	// for the values stored using that key:
	//
	// 	// Package user defines a User type that's stored in Contexts.
	// 	package user
	//
	// 	import "context"
	//
	// 	// User is the type of value stored in the Contexts.
	// 	type User struct {...}
	//
	// 	// key is an unexported type for keys defined in this package.
	// 	// This prevents collisions with keys defined in other packages.
	// 	type key int
	//
	// 	// userKey is the key for user.User values in Contexts. It is
	// 	// unexported; clients use user.NewContext and user.FromContext
	// 	// instead of using this key directly.
	// 	var userKey key
	//
	// 	// NewContext returns a new Context that carries value u.
	// 	func NewContext(ctx context.Context, u *User) context.Context {
	// 		return context.WithValue(ctx, userKey, u)
	// 	}
	//
	// 	// FromContext returns the User value stored in ctx, if any.
	// 	func FromContext(ctx context.Context) (*User, bool) {
	// 		u, ok := ctx.Value(userKey).(*User)
	// 		return u, ok
	// 	}
	Value(key interface{}) interface{}
}
```

