---
title: "Go Select 关键字"
date: 2020-11-10T22:55:44+08:00
draft: false
---

Select 关键字 default case 遇到的坑
<!--more-->

```Go
for {
    select {
        case <-a:
            do work1
        case <-b:
            do work2
        default:
    }
}
```

以上select, 原来希望是可以阻塞代码直到一个channel接收到数据。

然而，加入上面default的处理方式会不满足需求, 到出现不停执行for循环, 导致CPU占用过高。

因此，无需多此一举，添加不处理任何工作的default的判断。

```Go
for {
    select {
        case <-a:
            do work1
        case <-b:
            do work2
    }
}
```
