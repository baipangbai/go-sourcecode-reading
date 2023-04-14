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

**Context为规范，所有的初始都是这个规范。cancelCtx里面匿名了Context，所以cancelCtx可以直接被看成Context interface{}**

cancelCtx为主要的实现方，最重要的结构。





cancelCtx 默认初始化了一个，var cancelCtxKey int，里面有对cancelCtx的调用。

具体逻辑为：

```go
func (c *cancelCtx) Value(key interface{}) interface{} {
	if key == &cancelCtxKey {
		return c
	}
	return c.Context.Value(key)
}


parentCancelCtx中，有一行代码为 
`p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)`

这一行只要parent为cancelCtx类型，就会返回Context上下文中定义的cancelCtx
```

context中的设计，只想外暴露了规范，也就是Context interface{}，具体的实现为cancelCtx结构体和实现细节隐藏了。这种实现方式值得琢磨，为什么是否值得学习。



注意context.go的实现中，WithValue和WithCancel、WithDeadline不同

- WithValue底层的结构体为valueCtx
- WithCancel、WithDeadline底层结构体为cancelCtx



注意的是其中context，propagateCancel()中有一个新起goroutine来检测用户自定的Context是否关闭的逻辑。

```go
//下面代码可触发对应的逻辑。注意代码例子不完善，仅仅为了触发该逻辑。不能直接放到真实的业务环境运行，否则会出问题。

type MyContext struct {
}

func (*MyContext) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*MyContext) Done() <-chan struct{} {
	return make(chan struct{})
}

func (*MyContext) Err() error {
	return nil
}

func (*MyContext) Value(key interface{}) interface{} {
	return nil
}

func TestParentCtx(t *testing.T) {
	childCtx, childFun := WithCancel(&MyContext{})

	childFun()

	fmt.Println("test childCtx", childCtx)

	time.Sleep(3 * time.Second)
}
```



# 参考文档

- [ 内存泄漏](https://stackoverflow.com/questions/44393995/what-happens-if-i-dont-cancel-a-context)

- [官方使用案例和用法](https://go.dev/blog/context)

- [time.After example and notice](https://github.com/hellofresh/health-go/issues/89)
