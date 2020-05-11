---
layout: post
title:  "WebSocket"
categories: Web
tags: [JavaScript, WebSocket, 网络协议, 实时通讯]
---


WebSocket 是一个比较新的浏览器 API，有着诸多激动人心的特性。

参考 [RFC6455](https://tools.ietf.org/html/rfc6455) 规范。

## 概述

WebSocket 可以用来实现客户端和服务器的实时通讯，并且比较其他实时通讯技术，例如轮询，有着独特的优势，如实时性更好、网络开销更低等等。

WebSocket 使用 HTTP 协议建立连接，但并不是使用 HTTP 传输数据，而是在建立连接后，会切换到一个自定义协议，体现在 URL 上为：`ws://`、`wss://`（ 加密版，类似 HTTP 与 HTTPS 的关系 ）。

使用自定义协议带来了一个好处是，每次通讯可以发送更少的数据。


而使用 HTTP 请求的时候，单单首部字段带会引入很多流量消耗。一个主要的原因是，HTTP 请求是无状态的，服务器不会记住请求是谁发送的，每次请求都需要携带识别用的信息。

而 WebSocket 是可以记住状态的，于是可以使用自定义协议仅发送真正有意义的数据，从而节省大量的流量开销。

这在移动应用上特别重要。

WebSocket 分为握手和传输两部分。

其中握手阶段，是使用 HTTP 请求的，并有个协议升级的过程，可以从 HTTP 请求头中观察到这个过程，下面详细说明。


## 握手过程

握手阶段，使用 HTTP 请求。并且在 HTTP 请求的 Headers 中，会携带一些特别的字段。

### 握手请求的 Headers 例子

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13

```

### 服务器响应的 Headers 例子

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### Headers 字段解释

* `Connection: Upgrade` 及 `Upgrade: websocket`

  告知服务器，需要升级（切换）协议，切换成 WebSocket 协议。

* `Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==`

  该字段的用于防止 WebSocket 被未认证的浏览器脚本通过 WebSocket API  跨域使用。
  字段值是一个随机生成的 Base64 编码值（服务器响应中会有匹配字段来作为验证）。


* `Sec-WebSocket-Version: 13`

  用来告知服务器使用的 WebSocket 版本

* `Sec-WebSocket-Protocol: chat`

  可选字段，用来标识客户端可以接受哪个子协议。

* `HTTP/1.1 101 Switching Protocols`

  状态码 101 表示握手成功。

* `Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=`

  用来应答请求中的 `Sec-WebSocket-Key`。
  浏览器会验证该值，若不匹配，会抛出 `Error during WebSocket handshake` 错误。

  值是 `Sec-WebSocket-Key` 的值拼接上 GUID 、再 SHA-1 加密、最后 Base64 编码的结果。具体过程：

  1. 从请求头 `Sec-WebSocket-Key` 取得值，结果：  `dGhlIHNhbXBsZSBub25jZQ==`

  2. 拼接上 GUID `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`，结果：
  `dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11`

  3. SHA-1 加密处理，结果：
  `0xb3 0x7a 0x4f 0x2c 0xc0 0x62 0x4f 0x16 0x90 0xf6 0x46 0x06 0xcf 0x38 0x59 0x45 0xb20xbe 0xc4 0xea`

  4. Base64 编码处理，结果：
  `s3pPLMBiTxaQ9kYGzzhZRbK+xOo=`

客户端在脚本验证这些值，当响应的 HTTP 状态码为 101，`Sec-WebSocket-Accept` 的值符合预期，则可认为服务器已经建立连接。
接下来即可开始传输数据。


## 数据传输

WebSocket 连接建立之后，后续的通讯，都采用 WebSocket 协议。

这种协议无需每次传输都携带庞大的 Headers。
如果没有包含扩展，那么从客户端传输到服务器的数据，包含的头部只需要 2-10 字节 (和数据包长度有关)，反过来从服务器到客户端的数据传输，额外多4字节的掩码。


## 客户端实现

由于这是一个比较新的 API，因此使用过程需要留意客户端（我们只关注浏览器）的实现情况。

支持 WebSocket 的浏览器

* Chrome 4+
* Firefox 5+
* IE 10+
* Safari (iOS) 5+
* Android Brower 4.5+

主流浏览器厂商对客户端 WebSocket API 的实现基本一致，因此使用标准 HTML5 定义的 WebSocket 客户端的 JavaScript API 即可。
此外也可以使用业界满足 WebSocket 标准规范的开源框架，如 Socket.io。


## 使用 WebSocket

### 创建

直接传入 url，`new` 出一个 `WebSocket` 的实例即可。

```js
var ws = new WebSocket('ws://websocketexample.com')
```

### 发送数据

使用 WebSocket 实例的 `send()` 方法。

```js
ws.send('Hello WebSocket!')
```

### 关闭 WebSocket

使用 WebSocket 实例的 `close()` 方法。

```js
ws.close()
```

### 连接状态

WebSocket 不同的连接状态可以通过 readyState 判断。

* `0`: CONNECTING, 连接还未打开
* `1`: OPEN, 连接已经打开，可以开始通讯
* `2`: CLOSING, 连接正在关闭
* `3`: CLOSED, 连接已关闭


### 事件机制

当 WebSocket 连接的 readyState 变为 1 时，触发 `open` 事件，表示已经可以传输数据。

```js
ws.onopen = function(){
  // 连接已建立，可以发送数据
  ws.send('Hello WebSocket!')
}
```

当 WebSocket 连接的 readyState 变为 3 时，会触发 `close` 事件，表示连接已关闭。

```js
ws.onclose = function(event){
  // 连接已关闭
  console.log('WebSocketClosed!')
} 
```

当收到服务器数据时，会触发 `message` 事件。可以从事件对象中读取传输的数据。

```js
ws.onmessage = function(event){
  // 收到数据
  console.log(event.data)
}
```

当出现错误时，会触发 `error` 事件。

```js
ws.onerror = function(event){
  // 出错时
  console.log('WebSocketError!')
}
```


