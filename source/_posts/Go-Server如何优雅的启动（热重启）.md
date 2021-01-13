---
title : Go-Server如何优雅的启动（热重启）
date: 2020-12-28 18:00:12
categories: Golang
tags :
  - golang
  - gin
  - go
---
# Go-Server如何优雅的启动（热重启）

> 假设平台修改了配置文件只能重启才生效，但又不能影响正在访问的用户该如何是好呢～

A：负载均衡不就好了，我们一个一个来更新～

B：那你需要重启的节点刚好正在处理咋办呢？

A：那就等着没有请求了我们再更新～

B：................(🙄️)

## 一、分析问题

![image-20201228155113032](images/one.png)

A正在访问请求，必须正常返回才完成了请求的闭环; 当正在重启时我们需要将A用户的请求正常返回;

## 二、热重启

>当关闭一个正在运行的进程时，该进程并不会立即停止，而是会等待所有当前逻辑继续执行完毕，才会中断。

### 2.1 原理

1. 监听信号（USR2..）
2. 收到信号时fork子进程（使用相同的启动命令），将服务监听的socket文件描述符传递给子进程
3. 子进程监听父进程的socket，这个时候父进程和子进程都可以接收请求
4. 子进程启动成功之后，父进程停止接收新的连接，等待旧连接处理完成（或超时）
5. 父进程退出，重启完成

## 三、Golang实现

### 3.1 监听信号

|  信号   | 键值 |                       说明                       |
| :-----: | :--: | :----------------------------------------------: |
| SIGHUP  |  1   |                  挂起（hangup）                  |
| SIGINT  |  2   |           用户从键盘按^c键或^break键时           |
| SIGQUIT |  3   |               用户从键盘按quit键时               |
| SIGTRAP |  5   | 跟踪陷阱（trace trap），启动进程，跟踪代码的执行 |
| SIGKILL |  9   |                  杀死、终止进程                  |
| SIGTERM |  15  |         结束程序(可以被捕获、阻塞或忽略)         |

```go
func ListeningSignals() {
	ch := make(chan os.Signal, 1)
	// 监听指定信号
	signal.Notify(ch, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	for {
		sig := <-ch
		fmt.Printf("Get Signal is %v\n", sig)
		ctx, _ := context.WithTimeout(context.Background(), 20*time.Second)
		switch sig {
		case syscall.SIGINT, syscall.SIGTERM:
			// 停止监听信号
		case syscall.SIGHUP:
			//  重启
		}
	}
}
```

### 3.2 服务实现

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"net"
	"net/http"
	"os"
	"os/exec"
	"os/signal"
	"syscall"
	"time"
)

var (
	server        *http.Server
	listener      net.Listener
	gracefulChild = flag.Bool("graceful", false, "listen on fd open 3 (internal use only)")
)

func main() {
	flag.Parse()
	http.HandleFunc("/hello", Hello)
	server = &http.Server{Addr: ":8100"}

	var err error
	if *gracefulChild {
		f := os.NewFile(3, "") // FD 3 is special number
		listener, err = net.FileListener(f)
	} else {
		url := fmt.Sprintf("%s", server.Addr)
		listener, err = net.Listen("tcp", url)
	}
	if err != nil {
		fmt.Printf("Listening failed: %v\n", err)
		return
	}

	go func() {
		err = server.Serve(listener)
		if err != nil {
			fmt.Printf("server.Serve failed: %v\n", err)
		}
	}()
	ListeningSignals()
}

func Hello(w http.ResponseWriter, r *http.Request) {
	// 让程序等待
	time.Sleep(time.Second * 10)
	w.Write([]byte("调用完成～"))
}

func ListeningSignals() {
	ch := make(chan os.Signal, 1)
	// 监听指定信号
	signal.Notify(ch, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	for {
		sig := <-ch
		fmt.Printf("Get Signal is %v\n", sig)
		ctx, _ := context.WithTimeout(context.Background(), 20*time.Second)
		switch sig {
		case syscall.SIGINT, syscall.SIGTERM:
			// 停止监听信号
			signal.Stop(ch)
			// 优雅地关闭服务器而不中断任何工作
			server.Shutdown(ctx)
			return
		case syscall.SIGHUP:
			//
			err := RestartServer()
			if err != nil {
				fmt.Printf("graceful restart failed: %v\n", err)
			}
			return
		}
	}
}

func RestartServer() error {
	tl, ok := listener.(*net.TCPListener)
	if !ok {
		return fmt.Errorf("listener is not tcp listener")
	}
	f, err := tl.File()
	if err != nil {
		return err
	}

	args := []string{"-graceful"}
	cmd := exec.Command(os.Args[0], args...)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	cmd.ExtraFiles = []*os.File{f}
	return cmd.Start()
}
```

## 四、参考资料

[Graceful Restart in Golang](https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/)

[Golang中的热重启](https://cloud.tencent.com/developer/article/1388556)
