---
title: "atomic pointer in Go 1.19"
date: 2022-08-13T15:39:29+08:00
draft: false
---

本文也同样用于公司内部的技术分享。
<!--more-->
随着 Go 1.19 的发布，Go 语言团队升级 [sync/atomic](https://pkg.go.dev/sync/atomic@go1.19), 在其中加入了 `atomic.Pointer[T]` 的新类型，也是第一个作为全新 API 加入标准库，支持泛型的数据类型，受到 Go 社区用户的关注。同时，官方在 [Release Notes](https://tip.golang.org/doc/go1.19#mem) 中，也提及到这个 API 的加入，使得原子值使用更加简单。

# atomic.Value 的时髦"替身" (sleeky alternative)
## 特征
`atomic.Pointer` 是泛型类。与 `atomic.Value` 不同的是，从 Value 类中拿出的数据，需要进行断言，才能取出需要的值。而 Pointer 得益于泛型，直接能得到对应的数据类型。

```go
type ServerConn struct {
	Connection net.Conn
	ID         string
	Open       bool
}


func main() {
	aPointer := atomic.Pointer[ServerConn]{}
	s := ServerConn{ID: "first_conn"}
	aPointer.Store(&s)
    
	fmt.Println(aPointer.Load())

	aValue := atomic.Value{}
	aValue.Store(&s)
	conn, ok := aValue.Load().(*ServerConn)
	if !ok {
		panic("assert is not ok")
	}
	fmt.Println(conn)
}
```
输出：
```
&{<nil> first_conn false}
&{<nil> first_conn false}
```

## 实现比较
`Value` 的 Store 操作，会在运行时检查、确定其存储的实际类型，如果传入的是 nil 值的接口类型 (any) 数据，或者与上次 Store 的数据类型不同，则会 panic。
```go
func (v *Value) Store(val any) {
	if val == nil {
		panic("sync/atomic: store of nil value into Value")
	}
	vp := (*ifaceWords)(unsafe.Pointer(v))
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
	for {
		typ := LoadPointer(&vp.typ)
		if typ == nil {
			// Attempt to start first store.
			// Disable preemption so that other goroutines can use
			// active spin wait to wait for completion.
			runtime_procPin()
			if !CompareAndSwapPointer(&vp.typ, nil, unsafe.Pointer(&firstStoreInProgress)) {
				runtime_procUnpin()
				continue
			}
			// Complete first store.
			StorePointer(&vp.data, vlp.data)
			StorePointer(&vp.typ, vlp.typ)
			runtime_procUnpin()
			return
		}
		if typ == unsafe.Pointer(&firstStoreInProgress) {
			// First store in progress. Wait.
			// Since we disable preemption around the first store,
			// we can wait with active spinning.
			continue
		}
		// First store completed. Check type and overwrite data.
		if typ != vlp.typ {
			panic("sync/atomic: store of inconsistently typed value into Value")
		}
		StorePointer(&vp.data, vlp.data)
		return
	}
}

// ifaceWords is interface{} internal representation.
type ifaceWords struct {
	typ  unsafe.Pointer
	data unsafe.Pointer
}

```

Pointer 的同样操作，因会在编译期确定类型，直接存储指针即可。实现和调用简单不少。
```go
// Store atomically stores val into x.
func (x *Pointer[T]) Store(val *T) { StorePointer(&x.v, unsafe.Pointer(val)) }
```

# 示例
## 数据竞争
以下代码由于会在一定的时间内并发地读写，造成可能数据竞争。在 `go run` 命令中，加上 `-race` 参数，在代码执行过程中，检测可能会发生的数据竞争情况。
```golang
func ShowConnection(p *ServerConn) {
	for {
		time.Sleep(1 * time.Second)
		fmt.Println(p, *p)
	}

}
func main() {
	c := make(chan bool)
	p := ServerConn{ID: "first_conn"}
	go ShowConnection(&p)
	go func() {
		for {
			time.Sleep(3 * time.Second)
			newConn := ServerConn{ID: "new_conn"}
			p = newConn
		}
	}()
	<-c
}
```
运行输出
```
go run -race ./main.go
&{<nil> first_conn false} {<nil> first_conn false}
&{<nil> first_conn false} {<nil> first_conn false}
==================
WARNING: DATA RACE
Write at 0x00c00013e870 by goroutine 8:
  main.main.func1()
      .../main.go:53 +0x6a

Previous read at 0x00c00013e870 by goroutine 7:
  runtime.convT()
     ../go/src/runtime/iface.go:321 +0x0
  main.ShowConnection()
      .../main.go:41 +0x64
  main.main.func2()
      .../main.go:48 +0x39

Goroutine 8 (running) created at:
  main.main()
      .../main.go:49 +0x16e

Goroutine 7 (running) created at:
  main.main()
      .../main.go:48 +0x104
==================
&{<nil> new_conn false} {<nil> new_conn false}
&{<nil> new_conn false} {<nil> new_conn false}
```

## 使用 atomic.Pointer
将上文的指针，换成 `atomic.Pointer` 后，执行时，没有数据竞争的警告。
```go
func ShowConnection(p *atomic.Pointer[ServerConn]) {
	for {
		time.Sleep(1 * time.Second)
		fmt.Println(p.Load())
	}

}
func main() {
	c := make(chan bool)
	p := atomic.Pointer[ServerConn]{}
	s := ServerConn{ID: "first_conn"}

	p.Store(&s)

	go ShowConnection(&p)
	go func() {
		for {
			time.Sleep(3 * time.Second)
			newConn := ServerConn{ID: "new_conn"}
			p.Swap(&newConn)
		}
	}()
	<-c
}

```
输出
```
go run -race .\main.go
&{<nil> first_conn false}
&{<nil> first_conn false}
&{<nil> new_conn false}
&{<nil> new_conn false}
&{<nil> new_conn false}
```


# 总结
原子操作是互斥锁以外，另一种操作共享资源的方法。`atomic.Pointer` 的加入，使得对指针类型数据的操作友好易用。
在不方便使用新特性的情况下，`atomic.Value` 仍然是对复合数据进行并发原子操作的好选择。

# 参考
- [Atomic Pointers in Go 1.19](https://betterprogramming.pub/atomic-pointers-in-go-1-19-cad312f82d5b)
- [Go 1.19 Release Notes](https://tip.golang.org/doc/go1.19#mem)
- [What’s new in Go 1.19?](https://blog.carlmjohnson.net/post/2022/golang-119-new-features/)
