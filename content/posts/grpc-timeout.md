---
title: "gRPC KeepAlive 设置参数"
date: 2020-11-20T20:31:38+08:00
draft: false
tags: ["gRPC"]
---

# Keepalive
一开始使用gRPC的stream模式的时候，遇到网络波动，Recv阻塞没有接受到信息，在gRPC的默认设置下，是会长时间等待，造成假死的现象。

<!--more-->

这种情况下，需要使用gRPC的Keepalive机制，无论客户端与服务端哪一方出现网络波动，在一定时间内Ping没有得到回应, 就需要断开连接，程序内部处理尝试重连。

## Server

```golang
type ServerParameters struct {
    // MaxConnectionIdle is a duration for the amount of time after which an
    // idle connection would be closed by sending a GoAway. Idleness duration is
    // defined since the most recent time the number of outstanding RPCs became
    // zero or the connection establishment.
    MaxConnectionIdle time.Duration // The current default value is infinity.
    // MaxConnectionAge is a duration for the maximum amount of time a
    // connection may exist before it will be closed by sending a GoAway. A
    // random jitter of +/-10% will be added to MaxConnectionAge to spread out
    // connection storms.
    MaxConnectionAge time.Duration // The current default value is infinity.
    // MaxConnectionAgeGrace is an additive period after MaxConnectionAge after
    // which the connection will be forcibly closed.
    MaxConnectionAgeGrace time.Duration // The current default value is infinity.
    // After a duration of this time if the server doesn't see any activity it
    // pings the client to see if the transport is still alive.
    // If set below 1s, a minimum value of 1s will be used instead.
    Time time.Duration // The current default value is 2 hours.
    // After having pinged for keepalive check, the server waits for a duration
    // of Timeout and if no activity is seen even after that the connection is
    // closed.
    Timeout time.Duration // The current default value is 20 seconds.
}
```

服务端的设置，主要看 Time 和 Timeout 参数

Time 指服务端在Time时间内未接收到来自客户端的活动, 比如在stream的过程中，没有接收到数据，就会发送Ping，检测连接是否还正常。**默认为2小时**， 最好定制该参数缩短时间，及时发现及时重试。我在公司的项目设置 2 * time.Minute

Timeout 指在发送上面的 Ping 后，如果在 Timeout 的时间内客户端没有响应，服务端会关闭此连接。

## Client

```golang
type ClientParameters struct {
    // After a duration of this time if the client doesn't see any activity it
    // pings the server to see if the transport is still alive.
    // If set below 10s, a minimum value of 10s will be used instead.
    Time time.Duration // The current default value is infinity.
    // After having pinged for keepalive check, the client waits for a duration
    // of Timeout and if no activity is seen even after that the connection is
    // closed.
    Timeout time.Duration // The current default value is 20 seconds.
    // If true, client sends keepalive pings even with no active RPCs. If false,
    // when there are no active RPCs, Time and Timeout will be ignored and no
    // keepalive pings will be sent.
    PermitWithoutStream bool // false by default.
}
```

同样地，客户端也需要设置 Time 和 Timeout，保证连接能够及时重试。

客户端的默认 Time为 +inf, 这样的话客户端不会进行Ping的发送。我在项目中改为了 2 * time.Minute。这样两端都有基本的重试机制的保证。


## 简单实验

在服务端和客户端各自设置好 Keepalive 时间后，本地开启并互相连接上，然后尝试断网。在一段时间后，日志会输出以下错误:

```plain
code = Unavailable desc = transport is closing
```

说明 Keepalive 生效，并且因未接收到连接关闭了 gRPC 连接。

### 参考文档
[gRPC GitHub文档](https://github.com/grpc/grpc-go/blob/master/Documentation/keepalive.md)