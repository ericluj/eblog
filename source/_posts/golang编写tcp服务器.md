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



### server
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

### client
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