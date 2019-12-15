---
layout: post
title: "如何写代码才能处理读多写少的场景"
date: 2019-12-15
---

在互联网高并发场景下， 如何写代码才能handle住多数情况下读，少数情况写的的case？

第一个问题， 假设我们有一个int64的变量count， 再多线程并发访问的情况下，读这个变量的频率大，写这个变量的频率小，我们怎么写代码才能在代码正确的前提下让代码效率更高？

第一种写法：

```go
func wrongCountOp(c int) {
	var count int64
	wg := sync.WaitGroup{}

	for i := 0; i< c; i++ {
		wg.Add(1)

		go func() {
			defer wg.Done()
			count++
		}()
	}

	wg.Wait()
	fmt.Println(count)
}
```
结果证明，这么写压根就是错误的， 在golang中count++和count+=1是等价的，+=1操作并非是原子操作。

第二种写法：

```go
func mutexCountOp(c int) {
	var count int64
	var mux sync.RWMutex

	wg := sync.WaitGroup{}

	for i := 0; i< c; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			mux.Lock()
			count++
			mux.Unlock()
		}()
	}

	wg.Wait()
}
```
这次，结果对了，然后我们benchmark看看这个操作的性能。随着goroutine的数量的增加， 性能降低也在预见之内。
```go
BenchmarkMutexCount10-8       	  300000	      4763 ns/op
BenchmarkMutexCount100-8      	   50000	     37204 ns/op
BenchmarkMutexCount1000-8     	    5000	    293363 ns/op
BenchmarkMutexCount10000-8    	     500	   2823428 ns/op
```

第三种写法：
```go
func atomicCountOp(c int) {
	var count int64

	wg := sync.WaitGroup{}

	for i := 0; i< c; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomic.AddInt64(&count, 1)
		}()
	}

	wg.Wait()
}
```

看看性能表现。貌似比mutex版本稍微好一点。
```
BenchmarkAtomicCount10-8      	  300000	      4155 ns/op
BenchmarkAtomicCount100-8     	   50000	     29551 ns/op
BenchmarkAtomicCount1000-8    	    5000	    294506 ns/op
BenchmarkAtomicCount10000-8   	     500	   2864244 ns/op
```

第二种第三种写法目前均没有考虑read count的情况，根据二八定律，我们就假设read次数占80%， write占20%吧。添加一个reading方法， 然后看看性能表现：

```
mutexCountOp with reading:
BenchmarkMutexCount10-8       	     100	 672260325 ns/op
BenchmarkMutexCount100-8      	       2	 969319399 ns/op
BenchmarkMutexCount1000-8     	       1	1932470576 ns/op
BenchmarkMutexCount10000-8    	       1	11224598807 ns/op

atomicCountOp with reading:
BenchmarkAtomicCount10-8      	       1	1845034231 ns/op
BenchmarkAtomicCount100-8     	       2	1138883388 ns/op
BenchmarkAtomicCount1000-8    	       3	 601325334 ns/op
BenchmarkAtomicCount10000-8   	       1	2472123832 ns/op
```


为了对比更加直观， 我们把上面的数据转换为表格。从表格可见，添加了reading后，性能急剧下降。实际上，在写数据的时候， 读的性能下降也是很厉害的。

并发write数 | mutex | mutext with reading | atomic|atomic with reading
---|---|---|---|---
10 | 4763 | 672260325 |4155|1845034231
100 | 37204 |969319399 |29551|1138883388
1000|293363 |1932470576 |294506|601325334
10000|2823428 |11224598807 |2864244|2472123832


问题二，针对于复杂类型的变量，如何才能在读多写少的情况下，保证性能呢？
比如map。第一种办法，我们仍然可以使用mutex， 这种情况下， 性能并不会太好， 尤其是map里面存储了太多的数据的时候，每当写map的时候， 都会造成读map性能的急剧下降。在互联网应用读多写少的情况下， 这种方式就显得特别的不合适。这个时候，我们可以采用sync.Map来解决。 


```go
type Map struct {
	mu Mutex
	read atomic.Value
	dirty map[interface{}]*entry
	misses int
}

type entry struct {
	p unsafe.Pointer
}

type readOnly struct {
	m       map[interface{}]*entry
	amended bool // 当dirty包含m中没有的key的时候，才会为true。
}
```

sync.Map的核心原理：
根据key找value的时候， 我们首先会从read中读取，如果entry没有expunged(删除),则返回value值。 如果read能够读到并且不是amended，那么就认为读取完成了。 这个过程是没有加锁的。根据前面的benchmark， 可以知道没有加锁的read性能会比加锁的高不少。 如果dirty中包含了read中没有的key的时候,我们就需要加锁后从dirty中查找value了。当然访问dirty前需要加锁。

插入新值的时候，首先read这个值，如果read存在这个键，并且这个entry没有被标记删除，尝试直接进行原子写入,写入成功，则结束。如果写入失败，则这个value必定被其他gorotine删掉了。此时则加锁，如果不是amended，则说明dirty为nil，然后从read copy一份数据到dirty，然后更新dirty，amended也要置为true。

删除key的时候， 我们也是先read这个值，如果能够read到， 则直接把entry原子置为expunged。否则加锁，加锁后判断amended， 如果amended则从dirty中删除这个key。

可见，不论load/store/delete, 都是首先尝试从read到的value中进行原子操作， 这样就避免了加锁逻辑， 这样去掉了加锁逻辑后，就会使得read性能得到进一步的提升。 在write操作（insert/delete)中， 也是首先尝试进行无锁操作。只有在迫不得已的情况下，才会加锁从dirty中进行操作。 对于读多写少的case来说， 这种先无锁再加锁就能提升不少的性能。


