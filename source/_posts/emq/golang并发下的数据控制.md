---
title: golang并发下的数据控制
date: 2022-08-21 15:47:12
tags: emq
categories: 
- go
- emq
---

golang并发下的数据控制，主要使用lock和atomic。

<!-- more -->

## 一般情况
其实笔者在最开始没有并发编程经验的时候，常常会有一个疑问：为什么并发编程需要加锁？
我们先来看一个例子
``` go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	var num int64
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			num++
		}()
	}
	wg.Wait()
	fmt.Println(num)
}
```
上面的代码，按照我们的一般理解，对num进行10000次加1，最后的结果应该是10000才对。但是他与普通的for循环不同之处在于，我们使用了goroutine。

如果你跑一跑这段代码，那么你会发现结果不是10000，而是小于它的某个数字。

想一想的话其实不难理解，因为使用了goroutine，所以多个goroutine同时对num进行加1，可能协程a和协程b执行时候拿到的值相同（假如都是50），协程a和b各自加1都为51，所以他们俩的操作重复赋值。虽然进行了两次操作但是结果只增加了1。

## lock
golang提供了sync.Mutex，我们可以用它来进行Lock，这样多个goroutine同时执行的时候会去抢锁，只有获取到锁的那个goroutine才能往下执行，没有获取到锁的goroutine会被阻塞，直到获取到锁。这样就能够保证num同一时间是会被一个goroutine来进行操作，为此我们改写代码，它的执行结果会一直是10000。
``` go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	var mtx sync.Mutex
	var num int64
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			mtx.Lock()
			defer mtx.Unlock()
			num++
		}()
	}
	wg.Wait()
	fmt.Println(num)
}
```
虽然sync.Mutex为我们保证了变量的正确操作，但如果是读多写少的场景，多个同时读的goroutine也会阻塞等待。这个时候我们可以使用sync.RWMutex，它为我们提供了读锁RLock()和写锁Lock()。读锁可以被多个goroutine同时获取到，而写锁则必须等待读锁全部释放才能获取。

# atomic
## int64
golang为我们提供了一个atomic包，我们可以使用它来进行原子操作。这样我们不需要加锁的情况下，就能保证并发编程中不会有变量的冲突了。

使用atomic.AddInt64来进行加减，第一个参数传入num的指针。当我们获取num的值，可以使用atomic.LoadInt64。

``` go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var wg sync.WaitGroup
	var num int64
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomic.AddInt64(&num, 1)
		}()
	}
	wg.Wait()
	// 打印值为10000
	fmt.Println(atomic.LoadInt64(&num))

	// num是10000不是9999，不执行
	atomic.CompareAndSwapInt64(&num, 9999, 1)
	fmt.Println(atomic.LoadInt64(&num))

	// num是10000，将num设置为1
	atomic.CompareAndSwapInt64(&num, 10000, 1)
	fmt.Println(atomic.LoadInt64(&num))
}
```
atomic还为我们提供了一个先比较再设置的方法CompareAndSwapInt64，只有当old值判断相等，才会执行设置操作，否则返回false。

除此之外还有atomic.StoreInt64（保存），atomic.SwapInt64（保存并返回旧值）。

## 指针
如果基本的类型无法满足我们需求，我们可以使用atomic.Value，它可以用来存放指针。利用它可以进行任意值的原子存储与加载。

但我们需要注意两点：1.不能存放nil 2.存储第一个值后，只能再存储同类型的值。