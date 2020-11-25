---
title: groupcache源码分析4
date: 2020-03-14 21:25:12
tags: groupcache
categories: 
- go
- groupcache
---

singleflight.go文件代码不多，主要是提供了一个竞争执行方法，利用sync.WaitGroup来让重复的请求，进行等待，只实际执行一次，并将执行结果返回给所有等待的请求。算是一个并发的处理机制。

<!-- more -->

关于sync.WaitGroup的简单使用就是在创建一个任务的时候wg.Add(1), 任务完成的时候使用wg.Done()来将任务减一。使用wg.Wait()来阻塞等待所有任务完成。

1. call结构体表示实际的一个请求，包括返回值，错误，和wg。
``` go
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}
```

2. Group结构体表示一类工作，可以当成命名空间，对于同一个Group下的相同key请求我们只执行一次方法。m是map，对应保存请求key和他的call指针。
``` go
type Group struct {
	mu sync.Mutex
	m  map[string]*call
}
```

3. Do方法是本文件的核心，我们具体看下。
先判断key是否咋g.m中存在，如果存在，那么说明同一个key被多次请求了，我们得到call指针，调用Wait方法，进行阻塞等待，等到call指针执行完后，取得val和err返回即可。注意这里加锁的操作，防止取数据时，m被修改了。假如g.m[key]不存在，那么说明当前是key的第一个请求，new(call)返回指针并且c.wg.Add(1)，实际上在整个过程中，也只会Add这一次，将其放入map。等待fn()完成后，执行Done()方法，解除Wait的阻塞，将值返回给其他多次相同请求。最后从map中移除，收尾工作完成。
``` go
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```