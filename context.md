# CONTEXT源码研究背景

> 项目接手后，熟悉业务代码发现异常情况（[内存泄漏](https://stackoverflow.com/questions/44393995/what-happens-if-i-dont-cancel-a-context)），修复过程中发现代码没必要写那么复杂，就给优化了下。
>
> 优化后发现性能竟然有提升，所以想看看为什么性能有提升。

```go
//go version 1.16.3

//项目优化之前的代码（修复了没有cancel()，会导致内存泄漏问题），非超时
func BenchmarkContext(b *testing.B) {
	ctx := context.TODO()
	tmpCtx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
	//project before not add (defer cancel()),if not,memory leak
	defer cancel()

	intCh := make(chan int, 1)

	a := 11
	go func() {
		intCh <- double(a)
	}()

	var result int
	select {
	case result = <-intCh:
		if result%2 == 0 {
			// fmt.Println("fail")
			return
		}
	case <-tmpCtx.Done():
		// fmt.Println("timeout")
		return
	}
	// fmt.Println("result is", result)

}


//项目优化之后的代码，非超时
func BenchmarkTimeAfter(b *testing.B) {
	var result int
	intCh := make(chan int, 1)

	a := 11
	go func() {
		intCh <- double(a)
	}()

	select {
	case result = <-intCh:
		if result%2 == 0 {
			// fmt.Println("fail")
			return
		}
	case <-time.After(200 * time.Millisecond):
		// fmt.Println("timeout")
		return
	}
	// fmt.Println("result is", result)
}

//项目优化之前的代码，超时
func BenchmarkContextTimeout(b *testing.B) {
	ctx := context.TODO()
	tmpCtx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
	//project before not add (defer cancel()),if not,memory leak
	defer cancel()

	intCh := make(chan int, 1)

	a := 11
	go func() {
		time.Sleep(210 * time.Millisecond)
		intCh <- double(a)
	}()

	var result int
	select {
	case result = <-intCh:
		if result%2 == 0 {
			// fmt.Println("fail")
			return
		}
	case <-tmpCtx.Done():
		// fmt.Println("timeout")
		return
	}
	// fmt.Println("result is", result)
}

//项目优化之后的代码，超时
func BenchmarkTimeAfterTimeout(b *testing.B) {
	var result int
	intCh := make(chan int, 1)

	a := 11
	go func() {
		time.Sleep(210 * time.Microsecond)
		intCh <- double(a)
	}()

	select {
	case result = <-intCh:
		if result%2 == 0 {
			// fmt.Println("fail")
			return
		}
	case <-time.After(200 * time.Millisecond):
		// fmt.Println("timeout")
		return
	}
	// fmt.Println("result is", result)

}

//公用函数
func double(a int) int {
	return a * a
}



//压测结果
//非超时情况下，TimeAfter性能会比context.WithTimeout高出50%性能。
BenchmarkContext-8            	1000000000	         0.0000352 ns/op
BenchmarkTimeAfter-8          	1000000000	         0.0000229 ns/op


//可以看到在超时的情况下，TimeAfter的实现性能比context.WithTimeout，高出几百倍不止
BenchmarkContextTimeout-8     	1000000000	         0.2012 ns/op
BenchmarkTimeAfterTimeout-8   	1000000000	         0.0002625 ns/op
```

根据上面压测结果来看，确实使用timeAfter会带来较大的性能提升。那么问题来了：

1. context.WithTimeout适合什么样的场景使用
2. context.WithTimeout为何相比time.After性能差距这么大，具体如何实现的？哪里会多耗费性能？



三大主体：

```go
type Context interface{
  	Deadline() (deadline time.Time, ok bool)
  	Done() <-chan struct{}
  	Err() error
  	Value(key interface{}) interface{}
}

// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
  Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}

// A canceler is a context type that can be canceled directly. The
// implementations are *cancelCtx and *timerCtx.
type canceler interface{
 	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}
```

Context为规范，所有的初始都是这个规范



```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}


var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

// Background returns a non-nil, empty Context. It is never canceled, has no
// values, and has no deadline. It is typically used by the main function,
// initialization, and tests, and as the top-level Context for incoming
// requests.
func Background() Context {
	return background
}

// TODO returns a non-nil, empty Context. Code should use context.TODO when
// it's unclear which Context to use or it is not yet available (because the
// surrounding function has not yet been extended to accept a Context
// parameter).
func TODO() Context {
	return todo
}
```

# context.value()

```go
  //Packages that define a Context key should provide type-safe accessors
	// for the values stored using that key:
	
	// Package user defines a User type that's stored in Contexts.
	package user
	
	import "context"
	
	// User is the type of value stored in the Contexts.
	type User struct {...}
	
	// key is an unexported type for keys defined in this package.
	// This prevents collisions with keys defined in other packages.
	type key int
	
	// userKey is the key for user.User values in Contexts. It is
	// unexported; clients use user.NewContext and user.FromContext
	// instead of using this key directly.
	var userKey key
	
	// NewContext returns a new Context that carries value u.
	func NewContext(ctx context.Context, u *User) context.Context {
	 		return context.WithValue(ctx, userKey, u)
	}
	
	// FromContext returns the User value stored in ctx, if any.
	func FromContext(ctx context.Context) (*User, bool) {
	 		u, ok := ctx.Value(userKey).(*User)
	 		return u, ok
	}
```

[官方使用案例和用法](https://go.dev/blog/context)

```go
func handleSearch(w http.ResponseWriter, req *http.Request) {
    // ctx is the Context for this handler. Calling cancel closes the
    // ctx.Done channel, which is the cancellation signal for requests
    // started by this handler.
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    timeout, err := time.ParseDuration(req.FormValue("timeout"))
    if err == nil {
        // The request has a timeout, so create a context that is
        // canceled automatically when the timeout expires.
        ctx, cancel = context.WithTimeout(context.Background(), timeout)
    } else {
        ctx, cancel = context.WithCancel(context.Background())
    }
    defer cancel() // Cancel ctx as soon as handleSearch returns. 
}

//如果不使用cancel()，直接return，ctx不会回收吗？会导致内存泄漏？
//If you use WithCancel, the goroutine will be held indefinitely in memory. However, if you use WithDeadline or WithTimeout without calling cancel, the goroutine will only be held until the timer expires.
//This is still not best practice though, it's always best to call cancel as soon as you're done with the resource.



// The key type is unexported to prevent collisions with context keys defined in
// other packages.
type key int

// userIPkey is the context key for the user IP address.  Its value of zero is
// arbitrary.  If this package defined other context keys, they would have
// different integer values.
const userIPKey key = 0

//其中的type key int，新定义的类型在go中是int的别名还是什么，判断是否相等会怎么判断？
```

