---

date: 2020-07-08
title: "《Go语言编程》习题参考答案整理（持续更新）"
tags: ["Go基础", "Go语言编程"]

---

## 第8章 goroutine和通道

### 练习 8.1

原题目：

修改`clock2`来接收一个端口号，写一个程序`clockwall`，作为多个时钟服务器的客户端，读取每一个服务器的时间，类似于不同地区办公室的时钟，然后显示在一个表中。如果可以访问不同地域的计算机，可以远程运行示例程序；否则可以伪装不同的时区，在不同的端口上本地运行：

```bash
$ TZ=US/Eastern ./clock -port 8010 &   
$ TZ=Asia/Tokyo ./clock -port 8030 & 
$ TZ=Europe/London ./clock -port 8030 &
$ clockwall NewYork=localhost:8010 Tokyo=localhost:8030 London=localhost:8020
```

参考答案:

服务端代码`clock.go`:

```go
package main

import (
	"flag"
	"fmt"
	"io"
	"log"
	"net"
	"time"
)

var port = flag.Int("port", 8000, "listen port")

func main() {
	// 解析命令
	flag.Parse()
	address := fmt.Sprintf("localhost:%d", *port)
	fmt.Printf("address:%s\n", address)
	listener, err := net.Listen("tcp", address)
	if err != nil {
		log.Fatal(err)
	}
    // 处理请求
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Println(err)
			continue
		}
		go handleConn(conn)
	}
}

func handleConn(conn net.Conn) {
	defer conn.Close()
	for {
		_, err := io.WriteString(conn, time.Now().Format("15:04:05\n"))
		if err != nil {
			log.Fatal(err)
		}
		time.Sleep(1 * time.Second)
	}

}
```

客户端代码`clockwall.go`：

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net"
	"os"
	"strings"
	"time"
)

type clock struct {
	name string
	host string
}

func main() {
	// 解析参数
	if len(os.Args) == 1 {
		fmt.Fprintf(os.Stderr, "使用方式: clockwall NAME1=HOST1 NAME2=HOST2 ")
		os.Exit(1)
	}
	clocks := make([]*clock, 0)
	for _, a := range os.Args[1:] {
		fields := strings.Split(a, "=")
		if len(fields) != 2 {
			fmt.Fprintf(os.Stderr, "参数错误:%s\n", fields)
			os.Exit(1)
		}
		// 将客户端名称、地址封装在clock结构体中
		clocks = append(clocks, &clock{fields[0], fields[1]})
	}
	// 请求TCP服务器
	for _, c := range clocks {
		go c.dial()
	}

	// 主 goroutine 需要休眠，否则其它goroutine还没工作完程序就退出了
	for {
		time.Sleep(time.Minute)
	}

}

func (c *clock) dial() {
	conn, err := net.Dial("tcp", c.host)
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	for {
		c.watch(os.Stdout, conn)
	}
}

func (c *clock) watch(w io.Writer, r io.Reader) {
	b := make([]byte, 1024)
	n, err := r.Read(b)
	if err == io.EOF {
		os.Exit(0)
	}
	// 结果输出
	if err == nil && n > 0 {
		fmt.Fprintf(w, "%s:%s\n", c.name, string(b))
	}
}
```

### 练习 8.2

原题目：

实现一个并发的`FTP`服务器。服务器可以解释从客户端发来的命令，例如`cd`用来改变目录，`ls`用来列出目录，`get`用来发送一个文件的内容，`close`用来关闭连接。可以使用标准的`ftp`命令作为客户端，或者自己写一个。

参考答案：

（待补充）

### 练习 8.3

原题目：

在`netcat3`中，`conn`接口有一个具体的类型`*net.TCPConn`，它代表一个`TCP`连接。`TCP`连接由两半边组成，可以通过`CloseRead`和`CloseWrite`方法分别关闭。修改主`goroutine`，仅仅关闭连接的写半边，这样程序可以继续执行来输出来自`reverb1`服务器的回声，即使标准输入已经关闭。

回声服务器`reverb1.go`代码：

```go
// 回声服务器
package main

import (
	"bufio"
	"fmt"
	"log"
	"net"
	"strings"
	"time"
)

func main() {
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	for {
		// 监听请求
		conn, err := listener.Accept()
		if err != nil {
			log.Println(err)
			continue
		}
		// 处理请求
		go handleConn(conn)
	}
}

func handleConn(conn net.Conn) {
	defer conn.Close()
	input := bufio.NewScanner(conn)
	for input.Scan() {
		go echo(conn, input.Text(), 1*time.Second)
	}
	if input.Err() != nil {
		log.Fatal(input.Err())
	}
}

func echo(conn net.Conn, shout string, delay time.Duration) {
	fmt.Fprintln(conn, "\t", strings.ToUpper(shout))
	time.Sleep(delay)
	fmt.Fprintln(conn, "\t", shout)
	time.Sleep(delay)
	fmt.Fprintln(conn, "\t", strings.ToLower(shout))
}
```

回声客户端`netcat3.go`代码：

```go
// 回声客户端
package main

import (
	"io"
	"log"
	"net"
	"os"
)

func main() {
	addr, err := net.ResolveTCPAddr("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	conn, err := net.DialTCP("tcp", nil, addr)
	if err != nil {
		log.Fatal(err)
	}
	done := make(chan struct{})
	go func() {
		io.Copy(os.Stdout, conn)
		log.Println("Done")
		done <- struct{}{}
	}()
	// 将控制台输入复制到请求中
	mustCopy(conn, os.Stdin)
	// 仅关闭TCP连接的写
	conn.CloseWrite()
	// 通过channel来控制同步，只有后台goroutine工作完毕，主goroutine才结束
	<-done
}

func mustCopy(dst io.Writer, src io.Reader) {
	if _, err := io.Copy(dst, src); err != nil {
		log.Fatal(err)
	}
}
```



---

参考：

[《Go语言编程》书中源码](https://github.com/adonovan/gopl.io/)

[torbiak/gopl](https://github.com/torbiak/gopl)