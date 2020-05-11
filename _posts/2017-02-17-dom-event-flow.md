---
layout: post
title:  "DOM (Level 2) 事件流"
categories: JavaScript
tags: [JavaScript, DOM, Event]
---


闲暇无事复习下 DOM 事件流，顺便做做笔记记录一些重点。

先看一段简单的 HTML 代码：

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Document</title>
</head>
<body>
  <div id="div"></div>
</body>
</html>
```

我们在点击 `div` 的时候，而事件流是这样传播的：

## 1. 捕获阶段

最先开始的是捕获阶段。

按照 DOM Level 2 标准要求，事件会从 `document` 开始向内部传播，途径 `html`、`body`，最终在事件目标 (event target) `div` 之前停止。从而有机会在事件流到达目标之前捕获。

然而实际上是这样的：

1. IE9, Safari, Chrome, Opera, Firefox 会从 `window` 开始捕获事件。
2. IE9，Safari，Chrome，Firefox，Opera 9.5+ 等浏览器，捕获阶段不会在事件目标前停止，会继续传播到事件目标。


## 2. 处于目标阶段

事件流到达事件目标 `div` 时，就是第二个阶段。

DOM Level 2 事件标准中，处于目标阶段被视为是冒泡阶段的起点一部分，因此事件目标上要操作事件，应该绑定冒泡阶段的事件处理器。

然而实际上，对于前文提到的那些浏览器，绑定的捕获阶段的事件处理器也一样会执行，因此，事件目标上就有两个机会操作事件。


## 3. 冒泡阶段

标准中，事件流从事件目标 `div` 出发，往外传播，途径 `body`，`html`, 传播到 `document` 为止。

然而…  

1. IE5.5之前，冒泡会跳过 `html`，从 `body` 直接到 `document`。
2. IE9, Firefox, Chrome, Safari 会一直冒泡到 `window`。


## 事件对象

事件在传播过程，事件对象是个重要的对象，它包含一些有用的信息。下面介绍跟事件流相关的两个属性：

### event.target 事件目标

事件流在传播的过程，事件目标即为嵌套最深的那个节点，也就是例子中的 `div`。
事件对象中，有个 `target` 属性，即指向事件目标。

### event.currentTarget

事件流在传播过程，触发途径的节点上的事件处理器时，事件对象上的 `currentTarget` 即为当前流经的节点。  
如例子中，如果在 `body` 上绑定了处理器，那么，事件流经 `body` 触发处理器的时候，`currentTarget` 指向 `body`。


## 控制事件传播

* `.stopPropagation()`  
  事件对象上面有个 `stopPropagation()` 方法，可以阻止事件的传播。

* `.stopImmediatePropagation()`  
  事件对象上的 `stopImmediatePropagation()` 方法，除了阻止事件的传播，还可以阻止当前流经节点上绑定的其他事件处理器的执行。
