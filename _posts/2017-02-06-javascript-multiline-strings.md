---
layout: post
title:  "JavaScript 中的多行文本"
categories: JavaScript
tags: [JavaScript]
---

有时候，我们需要使用 JavaScript 构造一些 HTML 代码，如下：

```html
<div id="outer">
    <div id="inner">
        内容
    </div>
</div>
```

用 JavaScript 字符串来表示就是：

```js
const html = '<div id="outer"><div id="inner">内容</div></div>'
```

为了可读性和避免写错，就有强烈的写成带缩进的多行文本的需求。
现阶段，JavaScript 的多行文本，大家马上能想到的就是用 ES6 的字符串模板字面量来创建。

```js
const html = `
<div id="outer">
    <div id="inner">
        内容
    </div>
</div>
`
```

忆苦思甜，以前都是怎么做的呢？

可能大家还有印象，最笨拙的字符串拼接方法：

```js
var html = ''
+ '<div id="outer">'
+     '<div id="inner">'
+       '内容'
+     '</div>'
+ '</div>'
```

还有利用反斜杠的方式：

```js
var html = '\
<div id="outer">\
    <div id="inner">\
        内容\
    </div>\
</div>\
'
```

利用数组的方式：

```js
var html = [
'<div id="outer">',
'    <div>',
'        内容',
'    </div>',
'</div>'
].join('')
```

不走寻常路的利用读取函数内容的方式：

```js
function hereDoc(docFn) {
    return docFn.toString().replace(/^[^\/]+\/\*!?\s?/, '').replace(/\*\/[^\/]+$/, '')
}

var html = hereDoc(function(){/*!
<div id="outer">
    <div id="inner">
        内容
    </div>
</div>
*/})
```

在浏览器环境中，也可以直接在 HTML 中写模板代码，然后 JavaScript 读取内容：

```html
<script id="template" type="text/template">
    <div id="outer">
        <div id="inner">
            内容
        </div>
    </div>
</script>
```

```js
var html = document.getElementById('template').textContent
```

讲了这么多种书写多行字符串的方法，有什么用处呢？  
其实什么用都没，大部分场景，无脑使用 ES6 的模板字面量即可。

---

全文完
