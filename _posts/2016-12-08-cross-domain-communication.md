---
layout: post
title:  "常见跨域通讯技术总结"
categories: JavaScript
tags: [JavaScript, AJAX, 跨域]
---


## 背景

基于安全性的考虑，浏览器会遵守一个叫做同源策略的安全策略，默认情况下，XmlHttpRequest 对象只能访问其所在页面同一个域 (域名、协议、端口均相同) 的资源。然而，许多应用程序开发过程中，对跨域资源的访问也是很常见的需求。因此，掌握各种合理的跨域资源访问技术是很重要的。

下面列举一些常见的跨域技术。


## CORS ( Cross-Origin Resource Sharing )

W3C 推出的 CORS 规范，定义了访问跨源资源时，浏览器和服务器的沟通方式。


使用这种跨域资源访问技术，需要服务器、浏览器同时支持。现代浏览器的支持情况：

* Firefox 3.5+
* Safari 4+
* IE 10+
* iOS Safari
* Android Browser
* Chrome

> IE 8 开始引入的 XDomainRequest 也实现了一部分的 CORS 规范。  
XDomainRequest 的使用跟 XMLHttpRequest 略有不同，  
如 open 方法只支持两个参数（因为所有xdr请求都异步）、不能发送cookie、只支持 get、post方法等等。

CORS 具体实现是，通过 HTTP 首部字段进行交互。大致过程：

1. 浏览器发送请求时，携带一个 `Origin` 首部，字段值为请求页面的源信息 (协议、域名、端口)
2. 服务器接收到请求后，检查 `Origin` 首部字段的值是否在允许的范围，从而作出不同的处理。返回请求时，在响应请求的首部字段中，增加一个 `Access-Control-Allow-Origin` 字段，字段值同请求中的 `Origin` 字段的值。如果请求的是公共资源，也可以使用 `*` 作为字段值
3. 浏览器判断响应请求的 `Access-Control-Allow-Origin` 字段值是否匹配，不匹配或者没有这个首部字段，则拒绝这个跨域请求

此外，对于 cookie 的发送控制，需要设置其他首部字段，发送复杂请求时会使用 Preflighted Request 机制 (提前发送一个 OPTIONS 请求)，相关细节这里不赘述

### 例子：

#### 服务器 ( Node.js )

```js
require('http')
.createServer(function(req, res) {
  res.setHeader('Access-Control-Allow-Origin', '*')
  res.end('CORS')
})
.listen(3000)
```

#### 浏览器

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>CORS</title>
  <script src="jquery.js"></script>
</head>
<body>
  <script>
    $.ajax('http://127.0.0.1:3000')
    .done(function(data) {
      console.log(data) // 打印出 'CORS'
    })
  </script>
</body>
</html>
```

运行后，将看到控制台打印出 'CORS'



## JSONP ( JSON with padding )

JSONP 的命名很有迷惑性，但其并不是一种 JSON 相似数据交换格式，而是一种巧妙的与服务器交换数据的方式。

JSONP 利用了 HTML 元素的 src 属性可以跨域访问资源的特性。  
通过动态地创建 `<script>` 并设置其 `src` 为需要跨域访问资源的 url 来绕过同源策略限制。  
只需要服务器配合返回一份特定结构的 JavaScript 代码，便能达到我们跨域请求资源的目的。  
服务器返回的 JavaScript 代码结构符合这样的特点：

1. 是一个指定名称的函数的调用
2. 我们需要的数据（ 如 JSON ）作为该函数的参数

如此一来，只需要我们事先在本地定义好这个指定名称的函数，便可以在请求到数据后使用该函数做相应的数据处理了。

JSONP 原理简单、使用方便，并且具有非常优秀的兼容性，所以广受开发者欢迎。

但 JSONP 也有其缺点。  
比如判断 JSONP 是否已经失败了，能做的，基本只有设置超时检测，但这种检测受限于用户网速、带宽条件，难以做到尽善尽美。至于 HTML5 新增的 `<script>` 的 `onerror` 事件，还未得到浏览器支持。

此外，JSONP 使用过程可能受到其他域的恶意代码影响，因此需要谨慎判断其他域的安全性是否可靠。

### 例子

#### 浏览器

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>JSONP</title>
  <script src="jquery.js"></script>
</head>
<body>
  <script>
    $.ajax({
      url: 'http://127.0.0.1:3000/',
      dataType: 'jsonp',
      jsonpCallback: 'mycb',
      success: function(data) {
        console.log(data) // 打印 { hello : 'world' }
      }
    })
  </script>
</body>
</html>
```

#### 服务器（ Node.js ）  

```js
'use strict'
const http = require('http')
const url = require('url')
const data = { 'hello': 'world' }
http.createServer(function(req, res) {
  let params = url.parse(req.url, true)
  // 回调名称，本例子中为 mycb
  let cbName = params.query.callback
  // 输出回调包裹数据的形式: mycb({ hello: 'world' })
  let resStr = `${ cbName }(${ JSON.stringify(data) })`

  res.writeHead(200, { 'Content-type': 'text/plain' })
  res.end(resStr)
})
.listen(3000)
```


## window.postMessage

使用 HTML5 引入的 message API 进行跨域。

具体方式是使用 postMessage 发送消息，该方法允许来自不同源的脚本进行安全的、异步的通信。而接受消息的一方则注册一个message事件的处理器，然后就可以从事件对象中提取有用的信息进行处理。

不同于上面提到的方法可以跟服务器直接交换数据，使用 postMessage 只能在两个窗口 ( iframe ) 之间交换数据，需要一个窗口里使用 `iframe` 打开另一个窗口，或者用 `window.open()` 方法打开另外一个窗口，因为这样才能获取目标窗口的 `window` 对象引用。


### API 说明：

`otherWindow.postMessage(message, targetOrigin, [transfer])` 

对象 `otherWindow` 是接受方的 `window` 对象。所以发送消息需要以能得到接收方的 `window` 对象引用为前提。通常有这么些得到方式：

* iframe  
  `win = window.frames[0]`  
  `win = document.getElementsByTagName('iframe')[0].contentWindow`
* window.open 的返回值  
  `win = window.open(...)`
* 由哪个 window 打开的  
  `win = window.opener`
* message 事件处理器的事件对象的属性  
  `win = event.source`

参数 `message` 为要传递的消息，考虑到浏览器支持情况，最好使用字符串（ 尽管规范允许是 JavaScript 的任意基本类型或可复制的对象 ）

参数 `targetOrigin` 是目标窗口的源（ 即 "协议+主机+端口" 的形式，端口号后面还有内容的话会被忽略 ) 。该参数限制了消息只传递给指定的窗口，如果将该参数设置为 `*` ，则可以向任意窗口传递信息，如果将该参数设置成 `/`，则只传递给同源的窗口。

### 接收消息：
```js
window.addEventListener('message', messageHandle, false)
function messageHandle(event) {
  // event.origin 为消息发送方的 origin
  // 可以用来确认发送方是否可靠
  if (event.origin !== 'http://example.org') return

  // event.data 即发送过来的消息内容
  console.log(event.data)

  // event.source 是发送消息的 window 对象的引用
  // 可以用来回发消息实现双向通讯
  event.source.postMessage('收到了', event.orign)
}
```


## document.domain

这种方式适合在同一个域名下，不同子域名的页面之间进行交互。

页面之间资源互访的限制如下：

* 端口不同，不允许  
  `http://a.com:3000/a.js` & `http://a.com:3001/b.js`
* 协议不同，不允许  
  `https://a.com/a.js` & `http://a.com/b.js`
* 域名与对应的IP，不允许  
  `http://a.com/a.js` & `http://123.123.123.123/b.js`
* 子域名不同，不允许  
  `http://a.a.com/a.js` & `http://b.a.com/b.js`
* 域名不同，不允许  
  `http://a.com/a.js` & `http://b.com/b.js`
* 同协议、同域名、同子域名、同端口，允许  
  `http://a.com/a.js` & `http://a.com/b.js`  
  `http://a.com/a/a.js` & `http://a.com/b/b.js`

其中，域名相同，但子域名不同的情况，可以通过设置 `document.domain` 属性来解决。注意该值只能设置为自身层级或者更高层级（ 如可以从`a.a.com`设置成 `a.a.com` 或 `a.com` ），设置其他值会报错。

一般，只需要将 `document.domain` 都去掉子域名前缀即可，例如，`http://a.a.com/a.html` 和 `http://b.a.com/b.html` 都将 `document.domain` 设置为 `a.com`，这样这两个页面就能相互访问了。

### 例子

#### a.html

```html
<!-- 略 -->
<script>
document.domain = 'a.com' // 统一成 a.com
var ifr = document.createElement('iframe')
ifr.src = 'http://b.a.com/b.html'
ifr.style.display = 'none'
document.body.appendChild(ifr)
ifr.onload = function() {
    // 可以访问 b.html
    var doc = ifr.contentDocument || ifr.contentWindow.document
    console.log(doc.getElementsByTagName('title')[0].innerText) // b.html
}
</script>
<!-- 略 -->
```

#### b.html

```html
<!-- 略 -->
<title>b.html</title>
<script>
document.domain = 'a.com' // 统一成 a.com
</script>
<!-- 略 -->
```


## location.hash + iframe

该方式使用了 location.hash

### 大致过程：

1. 动态创建一个不可见的 `iframe`
2. 将 `iframe` 的 `src` 设置为服务器数据页面URL（ 相当于 `GET` 方式的请求 ）
3. 服务器输出的页面中包含一段修改 `window.location` 的脚本，修改为**与请求发起页面同源的 url，然后数据放在 hash 部分**
4. 请求发起页面，检测 `iframe` 的 hash 变化 (hashchange ／ 轮询等)，处理数据


#### 服务器核心代码

```js
// 将数据输出到请求页面同源的代理页 url 的 hash 部分
let resStr = `
<script>
  window.location = http://example.com/proxy.html#${ data }
</script>
`
```

#### 浏览器

```html
<script>
  function fetchData(url, callback) {
    var iframe = document.createElement('iframe')
    iframe.style.display = 'none'
    iframe.src = url
    iframe.onload = function() {
      callback(iframe.contentWindow.location.hash.slice(1))
      window.location.hash = ''
      document.body.removeChild(iframe)
    }
    document.body.appendChild(iframe)
  }

  // 获取数据
  fetchData('http://example.com/data', function(data) {
    console.log(JSON.parse(data))
  })
</script>
```

这种方式，需要注意 url 的总长度是有限的，取决于浏览器的实现。并且只能通过 iframe 发起 GET 请求。


## window.name + iframe

这种方式，与 `location.hash` + `iframe` 方案的思路基本是一致的。

利用的是 `window.name` 在页面刷新后，值也不会变化的特点，使用 `iframe` 发起请求，数据存放在 `window.name` 里面，然后 `iframe` 里面通过脚本切换到请求发起页面同源的 URL，这样请求发起页面就能成功读取到 `window.name` 里的数据。同样，`window.name` 的长度也有限制 (2MB ?)。


#### 服务器核心代码
```js
let resStr = `
<script>
  window.name = 'data...'
</script>
`
```


#### 浏览器

```html
<script type="text/javascript"> 
  function fetchData(url, callback) {
    var lockFlag = false
    var iframe = document.createElement('iframe')
    iframe.style.display = 'none'
    iframe.onload = function() {
      if (lockFlag) {
        callback(iframe.contentWindow.name)
        iframe.contentWindow.document.write('')
        iframe.contentWindow.close()
        document.body.removeChild(iframe)
      } else {
        // 成功load之后，上锁
        // 以免 iframe 在切换 URL 下次 onload 时
        // 又再次执行，以便直接进入数据处理
        lockFlag = true

        // 切换成同源的代理页（空页面即可），以便读取数据
        iframe.contentWindow.location = 'http://example/proxy.html'
      }
    }
    iframe.src = url
    document.body.appendChild(iframe)
  }

  // 获取数据
  fetchData('http://example.com/data', function(data) {
    console.log(JSON.parse(data))
  })
</script>
```


## WebSocket

WebSocket 不受跨域限制


## flash 方案

没研究，略…


