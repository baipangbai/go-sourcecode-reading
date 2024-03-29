# CONTEXT源码研究背景

> 项目接手后，熟悉业务代码发现疑似异常代码，排查后发现其实不会引起内存泄漏。但发现代码没必要写那么复杂，就给优化了下。
>
> 优化后发现性能竟然有提升，所以想看看为什么性能有提升。

WithCancel()返回的cancel()函数不调用会引起内存泄漏，但是WithTimeout()返回的cancel()即使不调用cancel()，timeout elapses后依然会调用cancel()。

为了规范，建议都普遍在函数结束后调用cancel()。避免看起来懵，还要记得有的调用有的不调用。

```go
//go version 1.16.3

//项目优化之前的代码（修复了没有cancel()，看起来像是有内存泄漏但是其实没有），非超时
func BenchmarkContext(b *testing.B) {
	ctx := context.TODO()
	tmpCtx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
	
  //project before not add (defer cancel())
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


//time.After()，调用后必须在timeout elapses后才会销毁，所以会存在程序都执行完毕了，timer还有大量的没有销毁
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

//use !delay.Stop(), if not timeout,timer destroy not util timeout elapses
func BenchmarkTimeAfterCancel(b *testing.B) {
	var result int
	intCh := make(chan int, 1)

	a := 11
	go func() {
		intCh <- double(a)
	}()
  
	delay := time.NewTimer(200 * time.Millisecond)
	select {
	case result = <-intCh:
    if !delay.Stop() {
			<-delay.C
		}
		if result%2 == 0 {
			// fmt.Println("fail")
			return
		}
  case <-delay.C:
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



# Context源码解读

```go
//三大主体
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

**Context为规范，所有的初始都是这个规范。cancelCtx里面匿名了Context，所以cancelCtx可以直接被看成Context interface{}**

cancelCtx为主要的实现方，最重要的结构体。

### cancelCtx

每次调用`WithCancel()`，都会在传递进来的`Context`基础上`newCancelCtx()`，生成`c（child）`，会将`Context`包在`cancelCtx`结构体中，且`newCancel()`生成的`cancelCtx`带有了各种源码中定义的方法。

源码还定义了一个`var cancelCtxKey int`，该定义默认使得cancelCtxKey有了初始值。

`p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}`

👆上面函数的`&cancelCtxKey`值，就是`var cancelCtxKey int`的初始值。这么定义该方法主要是为了返回直接`parent`（如果`parent`为`cancelCtx`类型的话）的`cancelCtx`值，方便后面直接调用`parent.(cancelCtx).children`等内部结构

`func (c *cancelCtx) Value(key interface{}) interface{} {
	if key == &cancelCtxKey {
		return c
	}
	return c.Context.Value(key)
}`

### Context

`context`源码中的设计，只向外暴露了规范，也就是`Context interface{}`，具体的实现为`cancelCtx`结构体，对外该实现细节隐藏了。

**这种实现方式值得学习，只对外暴露了规范。实现细节隐藏。内部交互也是基于interface{}，比如cancelCtx、ValueCtx**

注意`context.go`源码的实现中，`WithValue`和`WithCancel`、`WithDeadline`不同

- `WithValue`底层的结构体为`valueCtx`
- `WithCancel`、`WithDeadline`底层结构体为`cancelCtx`

### 一个特殊的地方

`propagateCancel()`中有一个新起`goroutine`来检测用户自定义的`Context`是否关闭的逻辑。主要是针对用户自定义实现了`Context`，而不是使用源码中定义的`Context`

```go
//下面代码可触发对应的逻辑。注意代码例子不完善，仅仅为了触发该逻辑。不能直接放到真实的业务环境运行，否则会出问题。

type MyContext struct {
	done chan struct{} // created lazily, closed by first cancel call
}

func (m *MyContext) Deadline() (deadline time.Time, ok bool) {
	return
}

func (m *MyContext) Done() <-chan struct{} {
	if m.done == nil {
		m.done = make(chan struct{})
	}
	d := m.done
	return d
}

func (m *MyContext) Err() error {
	return errors.New("mycontext err")
}

func (m *MyContext) Value(key interface{}) interface{} {
	return nil
}

func (m *MyContext) cancel() {
	close(m.done)
}

func TestParentCtx(t *testing.T) {
	myCtx := &MyContext{}
	childCtx, childCancelFun := WithCancel(myCtx)

	myCtx.cancel()

	fmt.Println("test childCtx", childCtx)

	time.Sleep(3 * time.Second)
	childCancelFun()
}
```



# 参考文档

- [ 内存泄漏](https://stackoverflow.com/questions/44393995/what-happens-if-i-dont-cancel-a-context)

- [官方使用案例和用法](https://go.dev/blog/context)

- [time.After example and notice](https://github.com/hellofresh/health-go/issues/89)
