---
title: "jas HijackWrite自定义返回格式"
date: 2020-02-18T19:03:18+08:00
description: ""
categories: []
toc: false
dropCap: true
displayInMenu: false
displayInList: true
draft: false
resources:
- name: featuredImage
  src: ""
  params:
    description: ""
---

## 简介

jas的Context中,通过为HijackWrite加入自定义的函数，可以用来改变返回的结构。

## 问题

原来的默认固定返回结构是：

```json
{"data":...,"error":...}
```

这样的结构如果需要更好的信息展示的话，肯定不太方便。

如果需要自定义的数据的话，则利用HijackWrite，自定义数据。

## 方法

在使用的jas.Router.Config中有:

```go
HijackWrite func(io.Writer, *Context) int
```

来看看在没有实现这个函数的时候，jas是如何处理返回的：

```go
    // context.go
    var resp Response
    resp.Data = ctx.Data
    if ctx.Error != nil {
        ctx.Status = ctx.Error.Status()
        resp.Error = ctx.Error.Message()
    }
    var written int
    if ctx.config.HijackWrite != nil {
        ctx.responseWriter.WriteHeader(ctx.Status)
        written = ctx.config.HijackWrite(ctx.writer, ctx)
    } else {
        jsonBytes, _ := json.Marshal(resp)
        if ctx.Callback != "" { // handle JSONP
            if ctx.written == 0 {
                ctx.ResponseHeader.Set("Content-Type", "application/javascript; charset=utf-8")
                ctx.responseWriter.WriteHeader(ctx.Status)
            }
            a, _ := ctx.writer.Write([]byte(ctx.Callback + "("))
            b, _ := ctx.writer.Write(jsonBytes)
            c, _ := ctx.writer.Write([]byte(");"))
            written = a + b + c
        } else {
            if ctx.written == 0 {
                ctx.responseWriter.WriteHeader(ctx.Status)
                written, _ = ctx.writer.Write(jsonBytes)
            } else if resp.Data != nil || resp.Error != nil {
                written, _ = ctx.writer.Write(jsonBytes)
            }
        }
    }
```

用户默认返回的都存在resp.Data中，error存放在resp.Error中，再解析成JSON数据。
**只要自定义结构，JSON解析后，用ctx.writer.Write函数写入即可**

```go
router.Config.HijackWrite = func(writer io.Writer, ctx *jas.Context) int {
        var written int
        if ctx.Error != nil {
            resp := &Response{
                Code: int32(ctx.Error.Status()),
                Desc: ctx.Error.Message(),
            }
            jsonBytes, _ := json.Marshal(resp)
            written, _ = writer.Write(jsonBytes)
        } else {
            written, _ = writer.Write([]byte(reflect.ValueOf(ctx.Data).String()))
        }
        return written
    }
```

这样的话，自定义下来的返回结构可以这样的：

```json
// 发现错误时返回
{"status":...,"error":...}

// 正常返回
{"data":...}
```

## 注意

jas已长时间未更新，推荐使用现在仍在维护、流行的Web框架。
