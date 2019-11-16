---
layout: post
title: "waitgroup 源码解析"
date: 2019-11-13
---
WaitGroup在Golang语言开发中使用非常平凡，但是WaitGroup内部实现是什么样子的呢？标准库中WaitGroup虽然仅仅141行代码， 里面的文章可是大有可为。

首先看waitGroup struct定义.
```go
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage
	// for the sema.
	state1 [3]uint32
}
```
这是什么鬼？ 完全看不懂。

问题1： noCopy是个啥？

这个问题交给https://www.jianshu.com/p/002152a35136来解答吧。
    
问题2： state1是个啥？


结合注释以及下面的state方法，我们可以知道，state1应该是64位的数字， 其中高32位是有多少个goroutine在运行， 低32位是有多少个goroutine阻塞在wait方法上。
但是golang中原子操作需要64位alignment，而32位编译器压根无法保证，所以就分配了12个字节（64 + 32位）， 其中aligned的8位代表state， 剩下的4位代表semaphore. 还是有些抽象。
下面我们根据操作系统的类型来分别解析这段的含义。

针对64位编译器， 我们知道64位cpu一次性可以提取64位数据，所以在golang中指针都是八个自己向前向后跳的。 比如指针p, p值为1的时候，指向内存地址为1的位置， 然而p的值为2的时候，1和2这两个内存地址之间有8个字节（64位）的距离。针对64位编译器，state1的结构为

state高位 | state低位 | sema
---|---|---

针对32位编译器，我们有有两种方式。

方式一： 如果state1地址正好能够被8整除，那么这种情况就和64位情况一样了。即
state高位 | state低位 | sema
---|---|---
方式二：如果state1不能够被8整除， 那么就是下面这种情况。
sema | state高位 | state低位
---|---|---


```go
// state returns pointers to the state and sema fields stored within wg.state1.
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}
```

Add逻辑怎么看不懂！

先是调用state方法拿到state和sema的指针，即statep和semap。根据我们前面的介绍，先是statep的高位加上delta，然后经过移位运算得到变量v，即正在运行（尚未Done）的goroutine数。变量w是state低位，指正在处于wait的goroutine数量。
如果没有处于wait状态的goroutine或者正在运行的goroutine数量>0，那么认为add方法就该返回了。 （panic情况属于错误使用waitgroup情况下才会触发，在此先跳过）。如果经过上面操作后， 已经没有正在运行的goroutine，那么处于wait状态的gorotine也就没有必要再继续等待了，直接操作sema的release释放资源。释放后，处于wait状态进行acquire操作的goroutine就可以继续执行下面的操作了。 

```go
// Add adds delta, which may be negative, to the WaitGroup counter.
// If the counter becomes zero, all goroutines blocked on Wait are released.
// If the counter goes negative, Add panics.
//
// Note that calls with a positive delta that occur when the counter is zero
// must happen before a Wait. Calls with a negative delta, or calls with a
// positive delta that start when the counter is greater than zero, may happen
// at any time.
// Typically this means the calls to Add should execute before the statement
// creating the goroutine or other event to be waited for.
// If a WaitGroup is reused to wait for several independent sets of events,
// new Add calls must happen after all previous Wait calls have returned.
// See the WaitGroup example.
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep // trigger nil deref early
		if delta < 0 {
			// Synchronize decrements with Wait.
			race.ReleaseMerge(unsafe.Pointer(wg))
		}
		race.Disable()
		defer race.Enable()
	}
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32)
	w := uint32(state)
	if race.Enabled && delta > 0 && v == int32(delta) {
		// The first increment must be synchronized with Wait.
		// Need to model this as a read, because there can be
		// several concurrent wg.counter transitions from 0.
		race.Read(unsafe.Pointer(semap))
	}
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	if v > 0 || w == 0 {
		return
	}
	// This goroutine has set counter to 0 when waiters > 0.
	// Now there can't be concurrent mutations of state:
	// - Adds must not happen concurrently with Wait,
	// - Wait does not increment waiters if it sees counter == 0.
	// Still do a cheap sanity check to detect WaitGroup misuse.
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// Reset waiters count to 0.
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false)
	}
}

```
Done方法很好理解，就是Add的反响操作。
```go
// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```
Wait方法中，我们可以看到大部分代码都是在for循环里面的，其中的含义是， 如果不满足条件，那么wait方法就会一值阻塞。 在for循环内部，我们可以看到， 一直在通过atomic.LoadUint64获取wg实例中的state的值。v，w变量和Add方法中操作逻辑和含义是一致的。 如果v==0， 那么就说明所有gorotine已经完成了，wait方法可以不再阻塞了，直接返回， 否则则通过CAS方法，将处于wait状态的goroutine数量+1.并且向sema执行acquire操作，如果acquire不到，就阻塞等待。
```go

// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep // trigger nil deref early
		race.Disable()
	}
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		w := uint32(state)
		if v == 0 {
			// Counter is 0, no need to wait.
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
		// Increment waiters count.
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			if race.Enabled && w == 0 {
				// Wait must be synchronized with the first Add.
				// Need to model this is as a write to race with the read in Add.
				// As a consequence, can do the write only for the first waiter,
				// otherwise concurrent Waits will race with each other.
				race.Write(unsafe.Pointer(semap))
			}
			runtime_Semacquire(semap)
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
	}
}
```


