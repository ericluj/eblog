---
title: golang编写tcp服务
date: 2022-07-21 22:40:00
tags: emq
categories: 
- go
- emq
---

使用golang编写一个tcp服务，包含server和client。

<!-- more -->

## server

golang中提供了net包，我们可以用它来编写一个tcp服务。
可以通过net.Listen方法来创建一个地址的监听，返回一个listener。
``` go
listener, err := net.Listen("tcp", "127.0.0.1:9000")
```
再循环调用listener的Accept()方法，这个方法会等待并获取一个客户端的连接conn，是一个net.Conn结构体。
``` go
for {
    conn, err := listener.Accept()
}
```
然后启动一个goroutine来处理获取到的连接conn，一般是使用一个for循环来不停读取数据。对于conn我们将其当作一个数据流来对他进行处理即可，循环从其中读取数据并根据'\n'来进行分割。
``` go
reader := bufio.NewReader(conn)

for {
    line, err := reader.ReadString('\n')
}
```
并且可以使用conn.Write方法来对客户端连接进行数据返回。
``` go
conn.Write([]byte(line))
```

**完整代码**
``` go
package main

import (
	"bufio"
	"fmt"
	"log"
	"net"
	"runtime"
	"strings"
	"sync"
)

func main() {
	tcpServer := &TCPServer{address: "127.0.0.1:9000"}
	err := tcpServer.Run()
	if err != nil {
		panic(err)
	}

	select {}
}

type TCPServer struct {
	address string
	wg      sync.WaitGroup
}

func (t *TCPServer) Run() error {
	listener, err := net.Listen("tcp", t.address)
	if err != nil {
		return err
	}

	for {
		conn, err := listener.Accept()
		if err != nil {
			if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
				// 如果是超时等临时错误，先暂停当前goruntine交出调度，时间片轮转到后再恢复后续操作
				runtime.Gosched()
				continue
			}

			if !strings.Contains(err.Error(), "use of closed network connection") {
				return fmt.Errorf("listener.Accept() error - %s", err)
			}

			break
		}

		t.wg.Add(1)
		go func() {
			handle(conn)
			t.wg.Done()
		}()
	}

	t.wg.Wait()

	return nil
}

func handle(conn net.Conn) {
	reader := bufio.NewReader(conn)

	for {
		line, err := reader.ReadString('\n')
		if err != nil {
			log.Println(err)
			return
		}

		log.Println("receive: " + line)

		conn.Write([]byte(line))
	}
}

```

## client

客户端可以使用 dialer.Dial方法来对server进行连接，方法的返回值也是一个net.Conn结构体，具体使用与server没有区别。
``` go
conn, err := dialer.Dial("tcp", "127.0.0.1:9000")
```
我们可以给dialer的Timeout字段赋值来设置请求的超时时间，默认是没有超时。

**完整代码**
``` go
package main

import (
	"fmt"
	"net"
	"time"
)

func main() {
	dialer := &net.Dialer{
		Timeout: time.Second * 2,
	}

	conn, err := dialer.Dial("tcp", "127.0.0.1:9000")
	if err != nil {
		panic(err)
	}

	for i := 0; i < 10; i++ {
		conn.Write([]byte(fmt.Sprintf("%d\n", i)))
	}

	conn.Close()
}

```