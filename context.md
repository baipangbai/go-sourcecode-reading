# context.Withtimeout 源码研究背景

> 项目接手后，熟悉业务代码发现异常情况，[内存泄漏](https://stackoverflow.com/questions/44393995/what-happens-if-i-dont-cancel-a-context)，修复的过程中发现其实代码没必要写那么复杂，就给优化了下，优化后发现性能竟然有提升，所以想看看为什么性能有提升。

```go
//公用函数
func newEngineCtx(ctx context.Context) (res context.Context ,err error) {
  return context.TODO(), nil
}

//修改之前，非原始项目代码，经脱敏后的简单逻辑
ctx := context.TODO()
tmpCtx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
engineCtxCh := make(chan *context.Context, 1)

go func() {
  engineCtx, err := newEngineCtx(ctx)
  if err != nil {
    engineCtxCh <- engineCtx
    return
  }
}

var engineCtx *context.Context
select {
  case engineCtx<-engineCtxCh:
		  if engineCtx == nil {
    			return 
  		}
  case <-tmpCtx.Done():
		  fmt.Println("tmpCtx.Done")
  		//项目之前没有加cancel，这块如果不加会引发内存泄漏问题
		  cancel()
}



//优化后的版本
ctx := context.TODO()
engineCtxCh := make(chan *context.Context, 1)
go func() {
  engineCtx, err := newEngineCtx(ctx)
  if err != nil {
    engineCtxCh <- engineCtx
    return
  }
}

var engineCtx *context.Context
select {
  case engineCtx = <-engineCtxCh:
		  if engineCtx == nil {
    			return 
  		}
  case <-time.After(200* time.Millisecond):
		  fmt.Println("tmpCtx.Done")
  		//项目之前没有加cancel，这块如果不加会引发内存泄漏问题
		  cancel()
}

```









# 之前看的本份（后面可能全部删除）

> 译作”上下文“，准确的说是goroutine的上下文，包含goroutine的运行状态、环境、现场等信息
>
> 并发安全，context 用来解决 goroutine 之间退出通知、元数据传递的功能**

```go
源码结构

type Context interface{
	Done() <-chan struct{}
	Err() error
	Deadline() (deadline time.Time, ok bool)
	Value(key interface{}) interface{}
}
```

注意点：再多说一点，Go 语言中的 server 实际上是一个“协程模型”，也就是说一个协程处理一个请求。例如在业务的高峰期，某个下游服务的响应变慢，而当前系统的请求又没有超时控制，或者超时时间设置地过大，那么等待下游服务返回数据的协程就会越来越多。而我们知道，协程是要消耗系统资源的，后果就是协程数激增，内存占用飙涨，甚至导致服务不可用。更严重的会导致雪崩效应，整个服务对外表现为不可用，这肯定是 P0 级别的事故。这时，肯定有人要背锅了。

goroutine作用：goroutine之间传递上下文，包括：取消信号、超时时间、截止时间、k-v等等

WithCancel

WithDeadline

WithTimeout

WithValue

注意：

1. 不要将 Context 塞到结构体里。直接将 Context 类型作为函数的第一参数，而且一般都命名为 ctx。
2. 不要向函数传入一个 nil 的 context，如果你实在不知道传什么，标准库给你准备好了一个 context：todo。
3. 不要把本应该作为函数参数的类型塞到 context 中，context 存储的应该是一些共同的数据。例如：登陆的 session、cookie 等。
4. 同一个 context 可能会被传递到多个 goroutine，别担心，context 是并发安全的。

[context源码解析](https://golang.design/go-questions/stdlib/context/cancel/)：这个自己还没有完全看透彻，需要回炉再看一遍，总结

# 重读context

context代码总共不到500行，但是实现的细节很值得反复推敲，尤其对接口的使用。

1. context cancel可以反复cancel，代码中判断了如果已经cancel过了，再次cancel，直接返回
2. 其中主要是对这个几个接口
   1. Context
   2. canceler
3. 主要的结构体 
   1. cancelCtx
4. 主要的函数
   1. WithCancel（WithDeadline，WithTimeout底层依赖）
   2. **propagateCancel** 将节点挂靠到其所属的父节点上