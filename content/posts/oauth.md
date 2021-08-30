---
title: "Oauth state 与 nonce"
date: 2021-08-03T22:51:07+08:00
draft: false
---
接触 Oauth 2.0 state & nonce 
<!--more-->

```
+--------+                                   +--------+
|        |                                   |        |
|        |---------(1) AuthN Request-------->|        |
|        |                                   |        |
|        |  +--------+                       |        |
|        |  |        |                       |        |
|        |  |  End-  |<--(2) AuthN & AuthZ-->|        |
|        |  |  User  |                       |        |
|   RP   |  |        |                       |   OP   |
|        |  +--------+                       |        |
|        |                                   |        |
|        |<--------(3) AuthN Response--------|        |
|        |                                   |        |
|        |---------(4) UserInfo Request----->|        |
|        |                                   |        |
|        |<--------(5) UserInfo Response-----|        |
|        |                                   |        |
+--------+                                   +--------+

```
上图来自官方文档。
## 名词解释
- RP: Relying Party, 业务提供方
- OP: OpenID Provider, OpenID 提供方


## 简单步骤
1. 用户在业务提供方点击授权登录，业务服务将用户重定向至授权服务
2. 用户输入在授权服务方注册的账号，登录授权服务，并同意授权信息至业务服务。
3. 授权服务将用户重定向回业务服务指定的回调页面，并带有授权服务生成的**授权码**
4. 业务服务带着该授权码与授权服务通信，请求用户信息。
5. 授权服务返回用户信息（ID Token），业务服务拿到用户信息后，授权登陆完成，开始正式服务用户。

## state
接下来谈谈安全方面需要注意的参数。</br>

业务服务提供方生成的，用于维持第1步和第2步的授权请求和回调的统一状态。防止 CSRF 攻击。通常在浏览器 cookie 绑定。

## nonce
将用户授权会话与 ID Token 绑定，缓解回显攻击。防止攻击者拿到授权码后登陆受害者系统。该参数保证每个授权码只使用一次。

## 公司项目中实现
基本授权过程这里不展开讲，主要将两个安全参数的实践。
- nonce。公司对接的 OpenID 服务，授权服务没有在 ID Token Claims 中回显 nonce，故无法验证该值。原来预定的计划是保存授权码 code 与 nonce 绑定，关系存储到 Redis 中。返回的 ID Token Claims 中如果匹配，则允许用户登录当前服务，并同时将该 nonce 绑定关系从 Redis 删除。

- state。state 会在第一步重定向中的 URL query 参数带到授权服务中，授权服务回调时，同样将该参数回显。服务端事先将 session 与 state 绑定，回调访问时先从 session 中检查 state 的值，值匹配才允许继续授权流程。

- 生成 nonce 和 state，都使用 go 的 crypto/rand，生成32位长度随机字符串，并用 URL 标准的 Base64 编码。

## 参考文档
[OAuth Replay Attack Mitigation ](https://medium.com/@benjamin.botto/oauth-replay-attack-mitigation-18655a62fe53)更完整地解释 OAuth 安全相关的问题。</br>
[OpenID Connect 官方文档](https://openid.net/specs/openid-connect-core-1_0.html)
