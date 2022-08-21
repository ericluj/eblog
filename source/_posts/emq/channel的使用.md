---
title: channel和select的使用
date: 2022-08-07 10:26:10
tags: emq
categories: 
- go
- emq
---

channel和select的基本使用场景。

<!-- more -->

channel在golang中主要是用于协程间的通信，是go语言中较为复杂的一部分。这一章笔者并不打算深入的去讲解它，而是列举出在emq项目编写过程中比较常见的用法。

## select
select和channel的使用往往是成对出现。它的使用和switch有相似之处，通过case来对channel的操作（读或写）作出判断，选取对应的分支执行。它的选择并不像if那样顺序，如果有多个条件同时判断成功，那么会**随机执行**。如果没有任何条件符合，那么它会默认执行default，**没有default的情况则会阻塞**。

## 协程A通知协程B启动
先来看一段代码，这段代码中Loop start会在main start之前执行。
``` go
package main

import (
	"fmt"
	"time"
)

func main() {
	go Loop()
	time.Sleep(time.Second)
	fmt.Println("main start")

	time.Sleep(5 * time.Second)
}

func Loop() {
	fmt.Println("Loop start")
}
```
如果希望在main start之后才执行Loop start，可以通过channel传递方法启动的消息，然后利用select来阻塞。
``` go
package main

import (
	"fmt"
	"time"
)

func main() {
	start := make(chan int)
	go Loop(start)
	time.Sleep(time.Second)
	fmt.Println("main start")
	start <- 1

	time.Sleep(5 * time.Second)
}

func Loop(start chan int) {
	select {
	case <-start:
	}
	fmt.Println("Loop start")
}
```

## 生产者消费者模式
启动三个goroutine，一个往channel写数据，两个从channel读数据，达到数据交互的目的。
``` go
package main

import (
	"fmt"
	"time"
)

func main() {
	msg := make(chan int)
	go Producer(msg)
	go Consumer1(msg)
	go Consumer2(msg)
	time.Sleep(5 * time.Second)
}

func Producer(msg chan int) {
	for i := 0; ; i++ {
		fmt.Printf("producer: %d\n", i)
		msg <- i
		time.Sleep(time.Second)
	}
}

func Consumer1(msg chan int) {
	for {
		select {
		case val := <-msg:
			fmt.Printf("consumer1: %d\n", val)
		default:
		}
	}
}

func Consumer2(msg chan int) {
	for {
		select {
		case val := <-msg:
			fmt.Printf("consumer2: %d\n", val)
		default:
		}
	}
}
```
打印结果为
```
producer: 0
consumer1: 0
producer: 1
consumer2: 1
producer: 2
consumer2: 2
producer: 3
consumer2: 3
producer: 4
consumer1: 4
```
需要注意的是，多个goroutine在读同一个channel的时候，写入的数据只会被其中一个给读到。

## 消息广播
在生产者消费者模式中我们知道了，同一条数据只会被一个goroutine给读到。那倘若需要通知所有goroutine的时候该怎么办呢？我们可以利用close(channel)的特性，一个已经被close的channel我们仍然可以读取到数据，但是只能读取到类型的零值。
``` go
package main

import (
	"fmt"
	"time"
)

func main() {
	c := make(chan int)
	go Loop1(c)
	go Loop2(c)
	time.Sleep(5 * time.Second)
	close(c)
	time.Sleep(100 * time.Second)
}

func Producer(msg chan int) {
	for i := 0; ; i++ {
		fmt.Printf("producer: %d\n", i)
		msg <- i
		time.Sleep(time.Second)
	}
}

func Loop1(close chan int) {
	for {
		select {
		case <-close:
			return
		default:
		}
		fmt.Println("Loop1...")
		time.Sleep(time.Second)
	}
}

func Loop2(close chan int) {
	for {
		select {
		case <-close:
			return
		default:
		}
		fmt.Println("Loop2...")
		time.Sleep(time.Second)
	}
}
```
上面代码两个Loop方法各输出5次后将不会再输出，因为读取到close信号，Loop方法执行了return。这里有一个细节需要注意的是，如果将return换为break，那么Loop方法不会结束，因为break只会退出select，不执行case往下的语句，并无法退出for循环。

## 定时器
time.NewTicker返回ticker，然后使用ticker.C来定时执行。
``` go
package main

import (
	"fmt"
	"time"
)

func main() {
	f := "2006-01-02 15:04:05"
	ticker := time.NewTicker(2 * time.Second)
	for {
		select {
		case <-ticker.C:
			fmt.Println(time.Now().Format(f))
		}
	}
}
```