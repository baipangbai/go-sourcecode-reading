# sync.Map源码研究背景

> 自己项目中使用了sync.Map，但是引起性能问题，后面改为管道方式，显著提升了性能。
>
> 使用sync.Map引发的问题
>
> 1. gc heap频繁发生
> 2. cpu优化后单机降低5%
>
> 因此自己脑海中就带来两个问题：
>
> 1. 为什么使用sync.Map后自己项目的性能 会降低
> 2. 自己使用方式是否错了？官方封装sync.Map主要适用场景是什么？应该怎么用？
>
> go版本：go1.16

# sync.Map 使用项目情况  & 优化后的结构

```go
//实践中使用项目，具体实现逻辑比较复杂，不适合公开。下面是使用简单代码，实现了项目中的逻辑。

//使用sync.Map 项目使用情况
func main() {
 	aList := make([]int, 100)
	for i := range aList {
		aList[i] = i
	}
  
  fmt.Println(getResult(aList))
}

func getResult(aList []int) (result map[int]int) {
  result = make(map[int]int)
  
  var syncMap sync.Map
  var wg sync.WaitGroup
  
  
	for _, a := range aList {
    wg.Add(1)
  	go func(t int){
      defer wg.Done()
    	syncMap.Store(t, double(t))
  	}(a)
	}
  
  wg.Wait()
  
  for _, t := range aList {
    val, exist := syncMap.Load(t)
    if !exist {
      continue
    }
    val, ok := val.(int)
    if !ok {
      continue
    }
 		result[t] = val   
  }
  return 
}

//改造版本（性能提升版本）
func getResultOptimize(aList []int) (result map[int]int) {
  result = make(map[int]int)
  ch := make(chan map[int]int)
  var wg sync.WaitGroup
  
  for _, a := range aList {
    wg.Add(1)
    go func(t int){
      defer wg.Done()
      ch <- map[int]int{t: double(t)}
    }(a)
  }
  
  go func(){
    wg.Wait()
    close(ch)
  }()
  
  for val := range ch {
    for a, b := range val {
      result[a] = b
    }
  }
  return 
}


//公用函数
func double(a int) int {
  return a * a
}
```

### 官方注释

官方给的两个主要应用场景：

1. when the entry for a given key is only ever written once but read many times, as in caches that only grow（多读少写，cache不断增长）
2. when multiple goroutines read, write, and overwrite entries for disjoint sets of keys. （多个goroutine读、改、写多个不相交的数据集）

In these two cases, use of a Map may significantly reduce lock contention compared to a Go map paired with a separate Mutex or RWMutex（只有在上面两种场景下，相比使用普通map和Mutex或者RWMutex的组合，sync.Map才会显著减少锁竞争）

**有限的应用场景，很多场景并不适合使用sync.Map，用错了场景反而会引起性能下降。**

# 源码分析

```go
//sync.Map源码定义主要结构

type Map struct {
  mu Mutex
  read atomic.Value //readOnly
  dirty map[interface{}]*entry
  misses int
}

type readOnly struct {
  m map[interface{}]*entry
  amended bool
}

type entry struct {
  p unsafe.Pointer
}
```

### 结构体具体字段说明

`mu` 涉及到dirty map操作时，进行加锁操作

`read`对readOnly进行存储数据，提供原子操作atomic.Value()，读取或者修改时可不加锁。注意该map中存储了（*存储后面又删除，且置换为expunge的状态值*），但是dirty中不会存储

> 将readOnly中曾经存在但是后期删除的值，标识为expunge的时机 => 存储了readOnly中不存在需要往dirty map中存储时
>
> 1. 将新值放入dirty中
> 2. 判断readOnly中是否存在曾经存在但是后期删除的值，如果存在，将值置换为expunge标识
> 3. 将readOnly中非expunge值全部存储到dirty中 **这个时候dirty中存储了新的数据和readOnly中之前所有非expunge数据，readOnly中也存储了之前的所有数据**

`misses`判断是否进行了频繁的读取操作，对应到missLocked()函数，**如果*misses* >= dirty len，将dirty map更新到readOnly.m中**，amended 置为false，dirty => nil，misses => 0（大概意思是dirty中的值都load读了一遍，dirty中大概率都是读的数据，放入专门的read map上，将原有dirty置空，这样可以避免使用锁mu）

`amended`true表明dirty map中存储了readOnly中没有存储的字段

### 函数具体执行

- Load()

  优先从readOnly中读取（不需要mu lock，从atomic.Value()中直接读取map，从中获取数据）

  readOnly中不存在且数据非expunge，且read.amended为true，使用mu lock，再次判断readOnly中不存在，从dirty中读取值，调用mislocked()

  > 这里面之前判断一次readOnly中不存在，但是后面加锁又判断一次readOnly是否存在
  >
  > 重复判断的原因在于第二次，会判断是否为expunge，如果为expunge，不仅会修改readOnly中的值，且会将值存储在dirty中，所以需要进行加锁。

- LoadOrStore （同Load()）

- LoadAndDelete

  判断是否在readOnly中，在直接删除返回

  不在，mu lock，再次判断不在readOnly中，不在readOnly中且amended=true（在dirty中），删除dirty数据，missLocked()返回

  > **其中删除为执行Delete()方法，不是将key对应的value值赋予nil**
  >
  > 错误做法`syncMap.Store("1", nil)`，sync.Map存储的是指向value的指针值，也就是指向nil的指针值并不为0x0
  >
  > 正确做法`syncMap.Delete("1")`，sync.Map delete操作为`atomic.CompareAndSwapPointer(&e.p, p, nil`为直接将e.p值置为nil（0x0），而不是指向nil的key值。

- Range

  优先判断readOnly中amended是否为true，为true，需要加锁从dirty中取值，并将dirty值放到readOnly中，amended置为false，dirty置为空

  如果amended为false，说明值readOnly中都有，直接从readOnly中取值。就不用加锁了。

### expunge数据

```go
func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}

/**
dirtyLocked 有可能会执行将readOnly.m中的nil值置换为expunged，dirty中不存储任何expunge的值。

readOnly.m为expunged，具体路径需要运行Delete。
**/

//什么情况下readOnly.m中会存储nil的值呢？又在什么情况下会将nil值更换为expunge标识呢？
//具体复现数据
var syncMap sync.Map
syncMap.Store("1", "1")
fmt.Println(syncMap.Load("1"))

syncMap.Delete("1")

syncMap.LoadOrStore("2", "4")

```

### readOnly和dirty的转换关键点

1. dirty中不存储expunge数据
2. dirty数据转换为readOnly，且dirty数据置换为空之后，再次存储数据
   - readOnly中不存在数据时，需要存储新数据时候，会将readOnly数据（非expunge）全部同步到dirty中
   - 新存储的值也存储到dirty中
   - amended置为true时，标识dirty存在数据，但是readOnly中没有

### readOnly都会做哪些操作

- Store

  更新的时候，如果readOnly存在且不是expunge，直接更新readOnly值

  如果readOnly存在，且是expunge，取消expunge标识，更新值，dirty中存储

- LoadOrStore *同Store*

  > 上面能看出来，只使用Store和Load，其实就是普通的map & Mutex 结合版本而已，加了一个dirty map => read map的逻辑。
  >
  > 主要的性能优化，也就是锁竞争。唯一可以体现的就是dirty map中的数据更新到read map后。下次load数据，不用mutex，直接从readOnly中读取就可以了。也就是写入后，频繁的读（每个数据平均超过一次）会降低锁竞争的次数。
  >
  > 其中amended用来标识，dirty中存在readOnly中没有的数据
  >
  > dirty中不存储expunge数据
  >
  > **当调用Store方法，且该新值在readOnly中不存在的时候，dirty为nil，（之前数据都被load一遍，值全部放到了readOnly中，dirty被置为nil）会将新值存在dirty中，且会将readOnly中非expunge数据全部写入dirty。**这步操作数据就会**冗余存储**。

### 为什么expunge数据不入dirty，但是readOnly中要存储？

没有必要再次存储一堆没用的数据，占用额外的内存，后面dirty数据都平均读取一遍及其以上后，dirty数据会重新覆盖readOnly数据，那个时候readOnly中存储的expunge标识数据也会被擦除。

# 性能对比

sync.Map和map & mutex压测对比 & ch代替方案

> [sync.Map，map & mutex性能对比参考文章](https://medium.com/@deckarep/the-new-kid-in-town-gos-sync-map-de24a6bf7c2c#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjU5NjJlN2EwNTljN2Y1YzBjMGQ1NmNiYWQ1MWZlNjRjZWVjYTY3YzYiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2NzY5NDUyNTYsImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjExNzcxMDE4ODcyNTA1Mzk5NDU0MiIsImVtYWlsIjoiYmFpcGFuZ2JhaUBnbWFpbC5jb20iLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiYXpwIjoiMjE2Mjk2MDM1ODM0LWsxazZxZTA2MHMydHAyYTJqYW00bGpkY21zMDBzdHRnLmFwcHMuZ29vZ2xldXNlcmNvbnRlbnQuY29tIiwibmFtZSI6InBhbmcgYmFpIiwicGljdHVyZSI6Imh0dHBzOi8vbGgzLmdvb2dsZXVzZXJjb250ZW50LmNvbS9hL0FFZEZUcDdSOVhteUJaNGtfanVBTlo4UnZZOVFIMU01T1BQUWI4MnppYW5vPXM5Ni1jIiwiZ2l2ZW5fbmFtZSI6InBhbmciLCJmYW1pbHlfbmFtZSI6ImJhaSIsImlhdCI6MTY3Njk0NTU1NiwiZXhwIjoxNjc2OTQ5MTU2LCJqdGkiOiIzNTk2MDE3ZmVlMTlmM2FjMTU1NzBlYzg5OTRkNzgzOTc4M2Y2ZDU1In0.IzrhjNgR71Yu2nYLlCNdRxOekLpY0kEJyQ2pQCTN6w3hCwaBgOpnFbZqWQ8bx3MswmpyUNu9Ew57H9R1aSyha_ZaxZSiZlkEhqUkDo4OYx-DwjEbBMDi1D_GCtx6If7BS8grEAVagQ5T4GF8B0OYTmjBZ7jt4apafBzH_r5pOutS7BfdzziCVSZn87p7faCVMH4YDfy0lFJJGurV1YMd3fNEEDAUq2vamM-JYY6iZasl_4G7UVmmz9c0fuLLOkLNp_g4yNYzo6Td-fRnI5jkJ7PX4IU4lU4us7t6FKmZ1_qBRcuQ6kUACrE3IP7MNAEieTeZFvSkIM-cqHSPFfKyCQ)
>
> [go bench使用介绍](https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go)

```go
//normal map & mutex type RegularIntMap struct {
type RegularIntMap struct {
	sync.RWMutex
	internal map[int]int
}

func NewRegularIntMap() *RegularIntMap {
	return &RegularIntMap{
		internal: make(map[int]int),
	}
}

func (rm *RegularIntMap) Load(key int) (value int, ok bool) {
	rm.RLock()
	result, ok := rm.internal[key]
	rm.RUnlock()
	return result, ok
}

func (rm *RegularIntMap) Delete(key int) {
	rm.Lock()
	delete(rm.internal, key)
	rm.Unlock()
}

func (rm *RegularIntMap) Store(key, value int) {
	rm.Lock()
	rm.internal[key] = value
	rm.Unlock()
}

func nrand(n int) []int {
	i := make([]int, n)

	for ind := range i {
		i[ind] = rand.Int()
	}
	return i
}

//store bench compare
func BenchmarkStoreRegular(b *testing.B) {
	nums := nrand(b.N)
	rm := NewRegularIntMap()
	b.ResetTimer()
	for _, v := range nums {
		rm.Store(v, v)
	}
}

func BenchmarkStoreSync(b *testing.B) {
	nums := nrand(b.N)
	var sm sync.Map
	b.ResetTimer()
	for _, v := range nums {
		sm.Store(v, v)
	}
}

//delete bench compare
func BenchmarkDeleteRegular(b *testing.B) {
	nums := nrand(b.N)
	rm := NewRegularIntMap()
	for _, v := range nums {
		rm.Store(v, v)
	}

	b.ResetTimer()
	for _, v := range nums {
		rm.Delete(v)
	}
}

func BenchmarkDeleteSync(b *testing.B) {
	nums := nrand(b.N)
	var sm sync.Map
	for _, v := range nums {
		sm.Store(v, v)
	}

	b.ResetTimer()
	for _, v := range nums {
		sm.Delete(v)
	}
}

//load bench compare
var globalResult int

func BenchmarkLoadRegularFound(b *testing.B) {
	nums := nrand(b.N)
	rm := NewRegularIntMap()
	for _, v := range nums {
		rm.Store(v, v)
	}

	currentResult := 0
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		currentResult, _ = rm.Load(nums[i])
	}
	globalResult = currentResult
}

func BenchmarkLoadRegularNotFound(b *testing.B) {
	nums := nrand(b.N)
	rm := NewRegularIntMap()
	for _, v := range nums {
		rm.Store(v, v)
	}
	currentResult := 0
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		currentResult, _ = rm.Load(i)
	}
	globalResult = currentResult
}

func BenchmarkLoadSyncFound(b *testing.B) {
	nums := nrand(b.N)
	var sm sync.Map
	for _, v := range nums {
		sm.Store(v, v)
	}
	currentResult := 0
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		r, ok := sm.Load(nums[i])
		if ok {
			currentResult = r.(int)
		}
	}
	globalResult = currentResult
}

func BenchmarkLoadSyncNotFound(b *testing.B) {
	nums := nrand(b.N)
	var sm sync.Map
	for _, v := range nums {
		sm.Store(v, v)
	}
	currentResult := 0
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		r, ok := sm.Load(i)
		if ok {
			currentResult = r.(int)
		}
	}
	globalResult = currentResult
}


//concurrent bench compare（实践中使用项目，具体实现逻辑比较复杂，不适合公开。下面是使用简单代码，实现了项目中的逻辑。）
func BenchmarkConcurrentSync(b *testing.B) {

	result := make(map[int]int)
	aList := make([]int, 100)
	for i := range aList {
		aList[i] = i
	}

	var syncMap sync.Map
	var wg sync.WaitGroup
	for _, a := range aList {
		wg.Add(1)
		go func(t int) {
			defer wg.Done()
			syncMap.Store(t, double(t))
		}(a)
	}
	wg.Wait()

	for _, t := range aList {
		val, exist := syncMap.Load(t)
		if !exist {
			continue
		}
		v, ok := val.(int)
		if !ok {
			continue
		}
		result[t] = v
	}
}

func BenchmarkConcurrentRegular(b *testing.B) {

	result := make(map[int]int)
	aList := make([]int, 100)
	for i := range aList {
		aList[i] = i
	}

	rm := NewRegularIntMap()
	var wg sync.WaitGroup
	for _, a := range aList {
		wg.Add(1)
		go func(t int) {
			defer wg.Done()
			rm.Store(t, double(t))
		}(a)
	}
	wg.Wait()
	for _, t := range aList {
		val, exist := rm.Load(t)
		if !exist {
			continue
		}
		result[t] = val
	}
}

func BenchmarkConcurrentCh(b *testing.B) {

	result := make(map[int]int)
	ch := make(chan map[int]int)
	var wg sync.WaitGroup

	aList := make([]int, 100)
	for i := range aList {
		aList[i] = i
	}

	for _, a := range aList {
		wg.Add(1)
		go func(t int) {
			defer wg.Done()
			ch <- map[int]int{t: double(t)}
		}(a)
	}

	go func() {
		wg.Wait()
		close(ch)
	}()

	for val := range ch {
		for a, b := range val {
			result[a] = b
		}
	}
	return
}
func double(a int) int {
	return a * a
}

/**压测对比
BenchmarkConcurrentSync-8        	1000000000	         0.0003446 ns/op
BenchmarkConcurrentRegular-8     	1000000000	         0.0001874 ns/op
BenchmarkConcurrentCh-8          	1000000000	         0.0001813 ns/op
//项目中的使用场景，sync.Map性能最差，其次自己构造的map & mutex，最优为ch替代用法(从压测看map & mutex 和 ch 效果差不多

BenchmarkConcurrentRegular-8     	1000000000	         0.0001915 ns/op
BenchmarkConcurrentCh-8          	1000000000	         0.0001569 ns/op
又压了一次，效果不稳定，不过在该使用场景下，性能都比sync.Map要高
)

BenchmarkDeleteRegular-8         	 9092030	       181.1 ns/op
BenchmarkDeleteSync-8            	 5493274	       256.0 ns/op
//存储后遍历一遍删除，不适合使用sync.Map

BenchmarkLoadRegularFound-8      	14966640	       132.0 ns/op
BenchmarkLoadSyncFound-8         	 6126120	       242.9 ns/op
//存储后，只load一遍，不适合使用sync.Map

BenchmarkLoadRegularNotFound-8   	16666842	       104.0 ns/op
BenchmarkLoadSyncNotFound-8      	 6664221	       244.3 ns/op
//存储后，found都不存在，不适合使用sync.Map

BenchmarkStoreRegular-8          	 5940949	       241.6 ns/op
BenchmarkStoreSync-8             	 1478697	       689.5 ns/op
//只Store，并不调用其他方法不适合使用sync.Map
**/


//注意的问题，轻易不要使用sync.Map，发现在一次写入，3次读取情况下，性能还不如自己map & mutex实现的性能
func BenchmarkMoreLoadRegularFound(b *testing.B) {
	nums := nrand(b.N)
	rm := NewRegularIntMap()
	for _, v := range nums {
		rm.Store(v, v)
	}

	currentResult := 0
	b.ResetTimer()

	loadFunc := func(bN int) {
		for i := 0; i < bN; i++ {
			currentResult, _ = rm.Load(nums[i])
		}
	}

	for i := 0; i < b.N; i++ {
		loadFunc(b.N)
	}

	globalResult = currentResult
}

func BenchmarkMoreLoadSyncFound(b *testing.B) {
	nums := nrand(b.N)
	var sm sync.Map
	for _, v := range nums {
		sm.Store(v, v)
	}
	currentResult := 0
	b.ResetTimer()

	loadFunc := func(bN int) {
		for i := 0; i < bN; i++ {
			r, ok := sm.Load(nums[i])
			if ok {
				currentResult = r.(int)
			}
		}
	}

	for i := 0; i < b.N; i++ {
		loadFunc(b.N)

	}

	globalResult = currentResult
}

BenchmarkMoreLoadRegularFound-8   	   10000	    447796 ns/op
BenchmarkMoreLoadSyncFound-8      	   10000	    718739 ns/op

//值得注意的是 即使在写入一次，后读取三次，读取N次依旧是自己通过map & mutex效率高

```

# 总结

> sync.Map的设计思想属于以空间换时间，double map，一个read map（no mutex），一个dirty map（mutex）。

理论上讲：

1. sync.Map降低并发的关键在于 readOnly和dirty的搭配。readOnly 用到了atomic.Value不需要使用锁。能显著降低锁竞争，提高性能。

1. 使用场景比较适合写入后，频繁读取的场景，能显著的降低锁竞争。
2. 在上面列出的项目中，自己使用错了场景，自己的业务场景（存储一次后，马上全部读取一遍，就结束了操作）不仅没有任何降低使用锁的竞争的成本，还多了一次原子更新操作（将dirty的值赋值给readOnly），也就是自己的项目并不适合使用sync.Map

实际的压测：

各种场景下sync.Map的性能均不如自己通过map & mutex实现的性能，所以慎用sync.Map。
