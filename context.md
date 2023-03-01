# CONtext源码研究背景

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

