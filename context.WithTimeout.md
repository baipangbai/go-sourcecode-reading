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

