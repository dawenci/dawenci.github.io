---
layout: post
title:  "DOM 节点集合及其活动(live)、静态(static)之分"
categories: JavaScript
tags: [JavaScript, DOM]
---

我们批量处理页面上元素的时候，需要先选择出对应的节点集合，常用的有以下 API

* `document.getElementsByClassName()`
  - 返回一个 HTMLCollection，且是**活动的**
* `document.getElementsByTagName()`
  - 返回一个 HTMLCollection，且**活动的**
* `document.getElementsByName()`
  - 返回一个 NodeList，且是**活动的**
* `document.querySelectorAll()`
  - 返回一个 NodeList，且是**静态的**

上面出现了两种 collection，一种是 HTMLCollection ，一种是 NodeList。
那么这两种集合有什么区别呢？


### NodeList
一个 NodeList 是一组节点(nodes)的集合。

### HTMLCollection
一个 HTMLCollection 是一组元素(elements)的集合。
有个 nameItem 方法是 NodeList 中没有的。

> HTMLCollection 是一个历史产物，标准设计新 API 不会再使用这个接口了。

说完两种集合，再来解释 live、static 的概念

### 活动的(live)、静态的(static)

一个活动的集合，意味着在文档改变的时候，会自动更新这个集合。
而一个静态的集合，则只是生成了一个选择时的快照( snapshot )，不会自动刷新。


看代码：

HTML:

```html
<!-- 省略前面 -->
<body>
  <p><p>
</body>
<!-- 省略后面 -->
```

JavaScript:

```js
var static = document.querySelectorAll('p') 
var live = document.getElementsByTagName('p')

// 集合初始都包含一个`p`
console.log(static.length) // 1
console.log(live.length) // 1

// 往`body`里插入一个 `p`
document.body.appendChild(document.createElement('p'))

// 活动的集合会自动刷新，静态的集合不变
console.log(static.length) // 1
console.log(live.length) // 2  
```

## live 特性可能带来的问题

因为文档的改变会反应到活动的集合上，所以我们在循环处理一个活动的集合的时候，必须非常小心，不要在循环体上插入符合选择条件的元素，否则可能出现死循环。

例子代码：

```js
var live = document.getElementsByTagName('p')
// 死循环
for (var i = 0; i < live.length; i += 1) {
  document.body.appendChild(document.createElement('p'))
}
```

