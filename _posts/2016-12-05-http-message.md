---
layout: post
title:  "HTTP 请求报文"
categories: Web
tags: [HTTP, 网络协议]
---


## HTTP 报文结构

### 起始行

报文第一行内容即为起始行，请求报文和响应报文中有所区别

#### 请求报文的起始行包含：

1. HTTP 动作（GET, POST, PUT, DELETE, PATCH, ...）
2. 操作的资源
3. HTTP 版本

例：

```
GET /example/example.jpg HTTP/1.1
```

#### 响应报文的起始行包含：

1. HTTP 版本
2. HTTP 状态码
3. HTTP 状态描述

例：

```
HTTP/1.1 200 OK
```


### 首部(headers)

首部内容紧接着起始行之后，每行代表一个首部字段。
一个 HTTP 报文可以拥有多个或者零个首部字段。

每个首部字段的结构是一个名值对，字段名后面跟着一个冒号，冒号之后是字段值。如：

```
Content-Type: text/html
```

最后，**首部以一个空行结束**（即使后面没有 body 部分也不可省略）。

### 主体(body)

首部的空行之后，就是报文主体，主体是可选的。
主体用来携带 HTTP 报文的负载，内容可以是任意文本、二进制数据（图像、音频、视频…等等等等）。

### 例子

#### HTTP 请求报文例子

```
GET /hello.text HTTP/1.1
Accept: text/*
Accept-Language: en
(注意这里是空行)
```

#### HTTP 相应报文例子

```
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 6
(注意这里是空行)
Hello!
```

## 部分 HTTP 首部字段含义

* Accept 
  该字段用来告知服务器客户端能接受什么媒体类型，字段值是一个用逗号分割的媒体类型列表（支持通配符*），如：
  ```
  Accept: text/*, image/jpeg
  ```

* Accept-Charset 字段
  该字段用来告知服务器客户端能处理哪些字符集，或者哪些是首选字符集。

* Accept-Encoding 字段
  该字段用来告知服务器客户端接受哪些编码方式

* Accept-Language 字段
  该字段用来描述接受哪种语言（或首选哪些语言）

* Age 字段
  该字段用来告诉接收端，响应已经产生了多久。HTTP/1.1 缓存必须每个响应都携带该首部字段。

* Allow 字段
  该字段用来告知客户端，特定资源支持哪些 HTTP 动作（GET,POST...）

* Authorization 字段
  该字段由客户端发出，用来携带身份验证信息。

* Catch-Control 字段
  值是一个可缓存性相关的缓存指令

* Connection 字段
  
* Content-Encoding
  报文 body 的编码方式，例如gzip

* Content-Language
  报文 body 使用的自然语言（汉语、英语、法语等等等）

* Content-Length
  报文 body 的内容字节大小。（HEAD请求中，不会发送body）

* Content-Location
  报文 body 对应的 URL （常用于将客户端重定向至新URL）

* Content-Type
  报文 body 的 MIME 媒体类型

* Cookie 
  请求中携带的首部，内容为Cookie信息

* Date
  请求、响应报文创建的时间

* Etag
  body 实体标记，用于标识资源

* Expires
  实体首部，表示响应实效的时间，在缓存方面应用

* Host
  请求首部，表示客户端想访问的具体主机名、端口号（信息来自请求的URL）。HTTP/1.1 对于没有Host首部的请求应该返回 400 Bad Request

* If-Modified-Since
  请求首部，用来发起条件请求，资源若在上次访问后未修改，则返回 304 Not Modified 而无需返回资源。否则当作非条件请求来处理。

* If-Unmodified-Since
  请求首部，服务器检测请求的日期值，只有在该日期值之后资源都没修改过，才会返回该资源。

* If-Match
  请求首部，用于发起条件请求，比较 Etag 实体标记，一致则返回该资源。

* If-None-Match
  请求首部，提供一个实体 Etag 标记列表，服务器资源的 Etag 跟该列表的标记都不匹配时，返回资源。

* Last-Modified
  实体首部，说明资源最后一次修改的时间（动态生成的资源则为生成的时间）

* Location
  响应首部，服务器可通过该首部将客户端导向某个资源的 URL ，该资源可能在上次请求后被移动过，也可能是响应中新创建的（如POST新创建资源后返回的相应中携带该首部指向新建的资源）。

* Pragma
  请求首部，携带一些指令，常用于控制缓存行为。

* Referer （规范中拼写错误，本应为Referrer，但也只能将错就错，LOL）
  请求首部，用于标记该请求是从哪个页面的超链接中过来的（存在隐私争议）

* Set-Cookie
  响应首部，用于设置Cookie

* User-Agent
  请求首部，提供用户代理的信息

* Vary
  响应首部，值是一个首部列表，表示服务器根据这些首部来确定响应什么内容给客户端



## 查看 HTTP 请求

前端开发过程，我们可以通过 Chrome 浏览器的开发者工具来查看请求的内容。

在 开发者工具的 Network 面板中，所有的请求都会罗列，点击对应的请求名字，即可展示一个请求的完整信息。这对于分析问题很有帮助。





