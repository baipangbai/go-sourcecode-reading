
# channel

# 基本概念

1. 不要依赖共享内存来进行通信，通过通信来共享内存。go 并发的哲学。依赖CSP模型，基于channel实现。
2. Go的并发原则：目标简单，尽量使用channel，把goroutine当作免费的资源，随便用。
3. go的有锁数据结构，CSP概念组成因子之一 
4. 什么是csp，通信顺序通信，并发编程模型。简单的说是两个独立的并发实体通过共享的通讯(channel)进行通信的并发模型˜

# 向channel发送数据是什么过程

1. 如果检测到channel是空的，当前goroutine会被挂起
2. 对于不阻塞的发送操作，如果channel未关闭并且没有多余的缓冲空间（说明：a. channel是非缓冲型的，且等待接受队列里面没有goroutine；b. goroutine是缓冲型的，但循环数组里面已经装满了元素）
3. 向一个非缓冲型的channel发送数据，从一个无元素的（非缓冲型或缓冲型但空）的channel接受数据，会导致一个goroutine直接操作另一个goroutine的栈（背离了gc操作机制，gc假设goroutine只操作自己的栈），这个时候需要写屏障。

```go
if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) || (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
	return false
}
//此处未加锁的判断没有看明白
```

# 从channel接收数据的过程是怎样的

1. 从一个nil的channel中接受数据不会panic
2. 其他暂且没有细看可以看，下面关闭channel的过程这些自己比较感兴趣

# 关闭channel

1. 关闭一个nil channel， pannic
2. 关闭一个closed的channel，panic
3. 发送队列里如果有goroutine出队列，panic

总结：close 逻辑比较简单，对于一个 channel，recvq 和 sendq 中分别保存了阻塞的发送者和接收者。关闭 channel 后，对于等待接收者而言，会收到一个相应类型的零值。对于等待发送者，会直接 panic。所以，在不了解 channel 还有没有接收者的情况下，不能贸然关闭 channel。

# 从一个关闭的channel依然能读出数据

```go
从一个有缓冲的 channel 里读数据，当 channel 被关闭，依然能读出有效值。只有当返回的 ok 为 false 时，读出的数据才是无效的。
func main() {
	ch := make(chan int, 5)
	ch <- 18
	close(ch)
	x, ok := <-ch
	if ok {
		fmt.Println("received: ", x)
	}

	x, ok = <-ch
	if !ok {
		fmt.Println("channel closed, data invalid.")
	}
}

运行结果：
received:  18
channel closed, data invalid.
```

# close(channel)

1. 已经关闭的channel 会直接panic
2. 被关闭的channel，不能再向其中发送内容，否则会panic
3. for循环的channel，channel关闭后，for循环会直接推出
4. len(ch) 获取元素数量 cap(获取容量) 编译的时候直接确定的，不是函数调用，直接取的hchan的field的(hchan就是源码中的结构体)
5. 如果close channel时，有sender goroutine挂在channel的阻塞发送队列中，会导致panic
6. 可以从closed的channel中接受值
7. 关闭nil的channel会panic
8. 不进行初始化，即不调用make来赋值的channel 称为 nil channel (var a chan int)
9. 一个sender，多个receiver，由sender来关闭channel，通知数据已经发送完毕
10. 读写一个nil channel都会被阻塞

# 关闭一个channel，那么容易panic，那么如何优雅的关闭channel呢？

> don’t close (or send values to) closed channels.

> 不要从一个 receiver 侧关闭 channel，也不要在有多个 sender 时，关闭 channel。
比较好理解，向 channel 发送元素的就是 sender，因此 sender 可以决定何时不发送数据，并且关闭 channel。但是如果有多个 sender，某个 sender 同样没法确定其他 sender 的情况，这时也不能贸然关闭 channel。

根据sender和receiver分为以下几种情况

1. 一个sender，一个receiver
2. 一个sender，N个receiver
3. M个sender，1个receiver
4. M个sender，N个receiver

1，2情况 sender直接关闭就好，困难在3，4情况。

> 在 Go 语言中，对于一个 channel，如果最终没有任何 goroutine 引用它，不管 channel 有没有被关闭，最终都会被 gc 回收。

3. 解决方案：the only receiver says “please stop sending more” by closing an additional signal channel。

解决方案就是增加一个传递关闭信号的 channel，receiver 通过信号 channel 下达关闭数据 channel 指令。

4. 解决方案：any one of them says “let’s end the game” by notifying a moderator to close an additional signal channel。

> 和第 3 种情况不同，这里有 M 个 receiver，如果直接还是采取第 3 种解决方案，由 receiver 直接关闭 stopCh 的话，就会重复关闭一个 channel，导致 panic。因此需要增加一个中间人，M 个 receiver 都向它发送关闭 dataCh 的“请求”，中间人收到第一个请求后，就会直接下达关闭 dataCh 的指令（通过关闭 stopCh，这时就不会发生重复关闭的情况，因为 stopCh 的发送方只有中间人一个）。另外，这里的 N 个 sender 也可以向中间人发送关闭 dataCh 的请求。

```go
//示例
func main() {
	rand.Seed(time.Now().UnixNano())

	const Max = 100000
	const NumReceivers = 10
	const NumSenders = 1000

	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})

	// It must be a buffered channel.
	toStop := make(chan string, 1)

	var stoppedBy string

	// moderator
	go func() {
		stoppedBy = <-toStop
		close(stopCh)
	}()

	// senders
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
					select {
					case toStop <- "sender#" + id:
					default:
					}
					return
				}

				select {
				case <- stopCh:
					return
				case dataCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	// receivers
	for i := 0; i < NumReceivers; i++ {
		go func(id string) {
			for {
				select {
				case <- stopCh:
					return
				case value := <-dataCh:
					if value == Max-1 {
						select {
						case toStop <- "receiver#" + id:
						default:
						}
						return
					}

					fmt.Println(value)
				}
			}
		}(strconv.Itoa(i))
	}

	select {
	case <- time.After(time.Hour):
	}

}
```

总结：只由一个sender或者yigereceiver来通知关闭channel，没有的话构造一个中间人

# channel发送和接收元素的本质是什么

> All transfer of value on the go channels happens with the copy of value.

# 什么情况下会引起资源泄漏

channel可能会引起goroutine泄漏。

泄漏原因是goroutine操作channel后，处于发送或者接收阻塞状态，而channel处于满或空的状态下，一直得不到改变，垃圾回收器也不会回收此类资源，进而导致goroutine一直处于等待队列中不见天日。

对于一个channel，如果没有任何goroutine引用了，gc会对其进行回收操作，不会引起内存泄漏。

# channel有哪些应用

1. 停止信号
2. 任务定时

    ```go
    select {
    	case <-time.After(100 * time.Millisecond):
    	case <-s.stopc:
    		return false
    }
    //等待 100 ms 后，如果 s.stopc 还没有读出数据或者被关闭，就直接结束。这是来自 etcd 源码里的一个例子，这样的写法随处可见。

    //定时执行某个任务，也比较简单：
    func worker() {
    	ticker := time.Tick(1 * time.Second)
    	for {
    		select {
    		case <- ticker:
    			// 执行定时任务
    			fmt.Println("执行 1s 定时任务")
    		}
    	}
    }
    //每隔 1 秒种，执行一次定时任务
    ```

3. 接耦生产方和消费方

    ```go
    func main() {
    	taskCh := make(chan int, 100)
    	go worker(taskCh)

        // 塞任务
    	for i := 0; i < 10; i++ {
    		taskCh <- i
    	}

        // 等待 1 小时 
    	select {
    	case <-time.After(time.Hour):
    	}
    }

    func worker(taskCh <-chan int) {
    	const N = 5
    	// 启动 5 个工作协程
    	for i := 0; i < N; i++ {
    		go func(id int) {
    			for {
    				task := <- taskCh
    				fmt.Printf("finish task: %d by worker %d\n", task, id)
    				time.Sleep(time.Second)
    			}
    		}(i)
    	}
    }
    //5 个工作协程在不断地从工作队列里取任务，生产方只管往 channel 发送任务即可，解耦生产方和消费方。
    ```

4. 控制并发数

    ```go
    var limit = make(chan int, 3)

    func main() {
        // …………
        for _, w := range work {
            go func() {
                limit <- 1
                w()
                <-limit
            }()
        }
        // …………
    }

    //构建一个缓冲型的 channel，容量为 3。接着遍历任务列表，每个任务启动一个 goroutine 去完成。真正执行任务，
    //访问第三方的动作在 w() 中完成，在执行 w() 之前，先要从 limit 中拿“许可证”，拿到许可证之后，才能执行 w()，并且在执行完任务，要将“许可证”归还。这样就可以控制同时运行的 goroutine 数。

    //还有一点要注意的是，如果 w() 发生 panic，那“许可证”可能就还不回去了，因此需要使用 defer 来保证。

    ```

---

# 重读channel

CSP概念：通过通信来共享内存，而不是通过共享内存来通信

channel 可以与 select、cancel、timeout等很好的结合使用

## channel底层数据结构什么样

> [https://speakerd.s3.amazonaws.com/presentations/10ac0b1d76a6463aa98ad6a9dec917a7/GopherCon_v10.0.pdf](https://speakerd.s3.amazonaws.com/presentations/10ac0b1d76a6463aa98ad6a9dec917a7/GopherCon_v10.0.pdf) 关于channal介绍，目前还未看

1. channel底层采用的是循环数组实现了队列
2. 其中recvq，sendq底层采用了双向链表 为什么用了双向链表
3. 创建channel，是调用了底层的makechan()函数，返回的是指针，因为管道在函数之间可以直接传递不用指针引用。
4. 新建一个chan后，内存分配在堆上

## 向channel发送数据的过程是怎么样的

1. 底层采用了chansend来发送
2. 如果向一个已经关闭的通道发送数据，会直接panic，但向一个nil但channel发送不会panic
3. 阻塞的数据，其实已经构造了单独的goroutine且变量绑定在上面，入了channel的recvq或者sendq里面了
4. 如果向一个channel非缓存，或者有缓存缓存为空的情况下，会直接向sender goroutine的变量直接copy到receiver goroutine的变量上，减少了一次内存拷贝，避免了先拷贝到channel然后再拷贝到receiver goroutine上。但是涉及到一个问题，就是违背了gc原则，goroutine只操作自己的栈空间内存，因为sender goroutine copy变量到 receiver goroutine的时候需要写屏障来避免问题

## 从channel接收数据的过程是怎么样的

1. 底层分别采用了chanrecv1，和chanrecv2来接收，一个返回是否关闭，一个不返回
2. 从一个阻塞，但是为nil的channel中接收数据，是会一直被挂起，除非close channel，但是close channel会导致panic
3. 一个closed的有缓冲的channel，buf有元素的情况下，是可以正常接收到元素的

## 如何优雅的关闭channel

1. 关闭一个nil的channel会panic
2. 关闭一个已经closed的channel会panic
3. 向一个已经closed的channel发送数据会panic

channel关闭共有集中情况 对于一个channel，如何没有任何goroutine引用了，gc会对其进行回收操作，不会引起内存泄漏，3， 4情况就是运用了该特性

1. 1 sender, 1 reciver
    1. 直接从sender端关闭
2. 1 sender, m receiver
    1. 直接从sender端关闭
3. m sender, 1 recevier
    1. 构造一个单独的stopCh，如果recevier要关闭的话，通知stopCh(close stopCh)，sender监测到stopCh关闭了，就不再向receiver发送数据
4. m sender, n receiver
    1. 构造一个中间人toStop，来关闭stopCh，sender和receiver都可以向toStop(中间人发送信号)，这样子stopCh只会关闭一次

## 什么情况下channel会内存泄漏

channel可能引发goroutine泄漏

goroutine操作channel后一直处于阻塞状态，而channel一直为空或满的状态一直得不到改变

## channel发送和接受元素的本质是什么

值拷贝
