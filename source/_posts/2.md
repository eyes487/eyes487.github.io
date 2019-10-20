---
title: 简单请求和复杂请求
categories: 前端
tag: Http 
date: 2019-10-08
---

当一个资源从与该资源本身所在的服务器不同的 `域`、`协议` 或 `端口` 请求一个资源时，资源会发起一个**跨域 HTTP 请求**。`跨域资源共享(CORS)` 是一种解决跨域的机制，它使用额外的 HTTP 头，允许服务器声明哪些源站通过浏览器有权限访问哪些资源。另外，规范要求，那些可能对服务器数据产生副作用的 HTTP 请求方法 (**复杂请求**），浏览器必须首先使用 `OPTIONS` 方法发起一个预检请求，从而获知服务端是否允许该跨域请求。

## 1.简单请求/复杂请求

同时满足下列三大条件，就属于简单请求

 * 请求方式只能是，`get`、`post`、`head`

 * http请求头限制这几种字段：`Accept`、`Accept-Language`、`Content-Language`、`Content-Type`、`Last-Event-ID`

 * Content-Type只能取：`application/x-www-form-urlencoded`、`nultipart/form-data`、`text/plain`

>其他请求都属于属于非简单请求



## 2.预检请求（Preflighted Requests）

Preflighted Requests是CORS中一种透明服务器验证机制。预检请求首先需要向另外一个域名的资源发送一个 HTTP `OPTIONS` 请求头，其目的就是为了判断实际发送的请求是否是安全的。


>简单跨域请求不会发送options（预检请求），复杂跨域请求会发送options


## 3.如何避免发送options请求

 * 如何避免发送options请求，就是尽可能使用简单请求啦~

 * 还可以通过给服务器设置请求头 `Access-Control-Max-Age`来避免发送options请求，浏览器一般会有一个默认值，但是都不长久，可以自己设置
```bash
Access-Control-Max-Age: 600 //即预检请求的结果缓存10分钟
```
`Access-Control-Max-Age`方法对完全一样的url的缓存设置生效，多一个参数也视为不同url。也就是说，如果设置了10分钟的缓存，在10分钟内，所有请求第一次会产生options请求，第二次以及第二次以后就只发送真正的请求了。




-------------如果以上内容有不对的地方，欢迎大家指正------------