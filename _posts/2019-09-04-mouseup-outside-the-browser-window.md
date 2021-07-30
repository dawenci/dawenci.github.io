---
layout: post
title:  "鼠标移出窗口外释放的事件触发现象"
categories: JavaScript
tags: [JavaScript, DOM, Event]
---

mouseup 之类的事件，鼠标移出窗口（或 iframe）时，释放点击，不会触发，除非事件绑定在 documentElement、document、window 上面。

例如：

```javascript
window.addEventListener('mouseup', () => { console.log('on mouse up(window, capture)') }, true)
document.addEventListener('mouseup', () => { console.log('on mouse up(doc, capture)') }, true)
document.documentElement.addEventListener('mouseup', () => { console.log('on mouse up(html, capture)') }, true)
document.body.addEventListener('mouseup', () => { console.log('on mouse up(body, capture)') }, true)
window.addEventListener('mouseup', () => { console.log('on mouse up(window)') }, false)
document.addEventListener('mouseup', () => { console.log('on mouse up(doc)') }, false)
document.documentElement.addEventListener('mouseup', () => { console.log('on mouse up(html)') }, false)
document.body.addEventListener('mouseup', () => { console.log('on mouse up(body)') }, false)
```


1. 当鼠标在页面中点击，然后直接释放点击时，控制台输出

```
on mouse up(window, capture)
on mouse up(doc, capture)
on mouse up(html, capture)
on mouse up(body, capture)
on mouse up(body)
on mouse up(html)
on mouse up(doc)
```



2. 当鼠标在页面中点击，然后移动到窗口外释放点击时，控制台输出

```
on mouse up(window, capture)
on mouse up(doc, capture)
on mouse up(html, capture)
on mouse up(html)
on mouse up(doc)
```

该行为可能对自己实现一些 drag & drop 逻辑时有一定的影响。
