---
title: "初见 signal.NotifyContext"
date: 2021-08-31T00:28:54+08:00
draft: true
---

试用 Go 1.16 于 os/signal 包加入的 signal.NotifyContext
<!--more-->

```go
import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"time"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, os.Kill)
	defer stop()

	exitChan := make(chan struct{})

	fmt.Println("wait for signal")

	// in another goroutine
	go func(c context.Context) {
		for {
			select {
			case <-c.Done():
				fmt.Println("receives done signal do graceful shutdown")
				close(exitChan)
				return
			case <-time.After(10 * time.Second):
				fmt.Println("stop goroutine after 10 seconds")
				close(exitChan)
				return
			}
		}
	}(ctx)

	<-exitChan
	fmt.Println("program exits")
}
```
输出：
```
wait for signal
// press Ctrl + C
receives done signal do graceful shutdown
program exits
```

向NotifyContext传入希望接收的系统信号，得到 context和 cancelFunc。

context 传入其他 goroutine 中，接收到系统的终止信号。context 关闭，订阅 Done 的程序，按收到 Done 信号处理。