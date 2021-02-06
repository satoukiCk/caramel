---
title: "Netlify换域名遇到的问题"
date: 2020-05-01T12:53:26+08:00
draft: false
toc: false
dropCap: false
displayInMenu: false
displayInList: true
description: "换个域名也有各种问题呢"
---

换个域名也有各种问题呢
<!--more-->


## 小小的琐事

最近Netlify[搬域名](https://community.netlify.com/t/changes-coming-to-netlify-site-urls/8918)了，会把*.netlify.com重定向至*.netlify.app。按照Netlify的说法，用户基本不需要任何工作，就能完成迁移。

但我访问自己博客的时候出现了无法访问，提示网页迁移到其他地址的问题。

尝试了一些方法

- 更换魔法工具
- 清缓存
- 重置网络设置

都没有成功。

随后使用同一网络下的手机访问却是正常的，意识到是自己网络的问题。

最后手动设置了电脑的DNS，过一会，居然成功了，切换回运营商自己的DNS，暂时也没什么问题。

## 反思

没想到迁移域名会发生出DNS解析问题这种事。Netlify在国内的环境下，还是不太方便的。**多写一些文章**，申请一个自己的域名。
