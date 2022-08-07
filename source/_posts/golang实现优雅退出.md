---
title: golang实现优雅退出
date: 2022-08-07 11:36:32
tags: emq
categories: 
- go
- emq
---

golang实现优雅退出，使用sync.WaitGroup和signal.Notify。

<!-- more -->

## sync.WaitGroup
通常编写代码时我们会启用多个goroutine来执行任务，为了保证在所有goroutine执行完毕后主程序才退出，可以使用sync.WaitGroup。它类似一个计数器，Add(1)方法来加1，Done()方法来减1，使用Wait()方法来阻塞程序（如果计数不为0则不会往下执行）。

先看一个不使用sync.WaitGroup的例子。
``` go
package main

import (
	"fmt"
)

func main() {
	go func() {
		fmt.Println("one...")
	}()

	go func() {

		fmt.Println("two...")
	}()

}
```

上面这段代码并不会打印出任何东西，因为主程序并不会等待goroutine全部执行完就退出了，然后我们改进代码。
``` go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("one...")
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("two...")
	}()

	wg.Wait()
}
```

上面的代码会输出
```
two...
one...
```
或
```
one...
two...
```
wg.Wait()会阻塞住程序的退出，直到所有goroutine执行完毕（one和two的顺序不一致，因为goroutine无法保证执行顺序）。

或许你会问，上面的代码不是已经可以实现优雅退出了吗，还需要signal.Notify做些啥呢？事实上，一个正常的程序更多的时候是一直在运行，直到收到退出信号（例如Ctrl+c），才终止程序。因此需要使用signal.Notify来监听退出信号。

## signal.Notify
signal.Notify可以用来监听程序信号，它的第一个参数是chan<- os.Signal，第二个参数是...os.Signal（可以传入多个需要监听的信号类型）。它会将收到的信号传递给channel c，因此我们可以通过读channel来阻塞程序，直到获取到退出信号再继续往下执行。
``` go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	fmt.Println("start...")
	c := make(chan os.Signal)
	signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
	<-c
	fmt.Println("exit...")
}
```
上面的代码启动后不会主动退出，必须执行ctrl+c才会退出。

## 优雅退出
通过对sync.WaitGroup和signal.Notify的组合使用，再配合上channel关闭，我们就可以实现优雅退出。
``` go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func main() {
	c := make(chan os.Signal)
	signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)

	var wg sync.WaitGroup
	exitChan := make(chan int)

	wg.Add(1)
	go func() {
		defer wg.Done()
		for {
			select {
			case <-exitChan:
				return
			default:
			}
			fmt.Println("one...")
			time.Sleep(time.Second)
		}

	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		for {
			select {
			case <-exitChan:
				return
			default:
			}
			fmt.Println("two...")
			time.Sleep(time.Second)
		}
	}()

	<-c
	close(exitChan)
	wg.Wait()
}
```
收到退出信号后，通过close(exitChan)来通知所有goroutine结束for循环，wg.Wait()阻塞等所有goroutine结束。