---
title: "初见 signal.NotifyContext"
date: 2021-08-31T00:28:54+08:00
draft: false
---

试用 Go 1.16 于 os/signal 包加入的 signal.NotifyContext
<!--more-->

## 尝试 signal.NotifyContext
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

向NotifyContext传入希望接收的系统信号，得到 context 和 CancelFunc。

context 传入其他 goroutine 中，接收到系统的终止信号。context 关闭，接收 Done 的程序，收到进行处理相应。这样的信号处理，能很好的融合到代码中。


## 老的做法
以前的代码，是先接收信号，接收到信号后，调用相关的结束方法。
```go
	sig := make(chan os.Signal, 1)
	signal.Notify(sigChan,
		syscall.SIGKILL,
		syscall.SIGSTOP,
		syscall.SIGTERM,
		syscall.SIGQUIT,
	)

	<-sig
	
	someJob.done()
```


## 探索源代码
```go
// NotifyContext returns a copy of the parent context that is marked done
// (its Done channel is closed) when one of the listed signals arrives,
// when the returned stop function is called, or when the parent context's
// Done channel is closed, whichever happens first.
//
// The stop function unregisters the signal behavior, which, like signal.Reset,
// may restore the default behavior for a given signal. For example, the default
// behavior of a Go program receiving os.Interrupt is to exit. Calling
// NotifyContext(parent, os.Interrupt) will change the behavior to cancel
// the returned context. Future interrupts received will not trigger the default
// (exit) behavior until the returned stop function is called.
//
// The stop function releases resources associated with it, so code should
// call stop as soon as the operations running in this Context complete and
// signals no longer need to be diverted to the context.
func NotifyContext(parent context.Context, signals ...os.Signal) (ctx context.Context, stop context.CancelFunc) {
	ctx, cancel := context.WithCancel(parent)
	c := &signalCtx{
		Context: ctx,
		cancel:  cancel,
		signals: signals,
	}
	c.ch = make(chan os.Signal, 1)
	Notify(c.ch, c.signals...)
	if ctx.Err() == nil {
		go func() {
			select {
			case <-c.ch:
				c.cancel()
			case <-c.Done():
			}
		}()
	}
	return c, c.stop
}
```
基于传入的 context 和信号，创建 signalCtx 数据结构:
```go
type signalCtx struct {
	context.Context

	cancel  context.CancelFunc
	signals []os.Signal
	ch      chan os.Signal
}

func (c *signalCtx) stop() {
	c.cancel()
	Stop(c.ch)
}
```
stop 为 CancelFunc。

NotifyContext 最终还是调用 Notify 监听信号。接收到信号后， context 会调用 cancel 方法， 所有调用该 context.Done 的方法，都会执行响应的语句。

## 总结
NotifyContext 将 context.Context 和 os.Notify 进行封装。方便开发人员结合 Context.Done，编码处理系统信号的相关代码。

## 参考文章
- [Graceful Shutdowns in Golang with signal.NotifyContext](https://millhouse.dev/posts/graceful-shutdowns-in-golang-with-signal-notify-context)
- [Official doc](https://pkg.go.dev/os/signal#NotifyContext)