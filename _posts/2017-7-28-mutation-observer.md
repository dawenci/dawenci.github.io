---
layout: post
title:  "使用 MutationObserver 监视 DOM 变化"
categories: JavaScript
tags: [JavaScript, DOM]
---

MutationObserver 可以用来监控 DOM 的变动。
这个 API 定义在 DOM4 中，被设计来替代已经废弃的 DOM3 事件：Mutation Events。

该 API 与事件不同的是，它并不会在每个 DOM 节点变化后立即执行回调函数，而是在 DOM 操作都完成后，将所有变化记录存储在数组中，在事件循环的 microtask 阶段执行回调函数，一次性处理这些存储的变化。

> 注： 一次事件循环，从 macrotask quene 中取出一个任务执行，执行完毕后，执行整个 microtask quene 中的任务。

## 构造函数：

```js
MutationObserver(callback)
```

`callback` 在每个 DOM 变动中被调用，这个回调函数接受两个参数：

1. mutationRecords 改变记录列表
2. observer 即 MutationObserver 实例

<!-- more -->

### MutationRecord

MutationRecord 用以描述一个 DOM 变动的详细信息，它拥有如下字段：

* `type` {String}  
  返回描述改变的类型，值可能为：  
    - `attributes` 变化的为节点的 attribute
    - `characterData` 变化的为节点的文本值
    - `childList` 变化的为节点增删
  
  >  注：CharacterData 是一个抽象接口，表示node节点包含的文本数据。如文本节点、注释节点等实现了这个接口。

* `target` {Node}  
  视 `type` 字段的不同有不同结果：
    - attributes：值为被改变的节点
    - characterData：值为 CharacterData 节点
    - childList：值为包含这些改变子节点的父节点

* `addedNodes` {NodeList}  
  返回新增的节点，无新增则返回空节点列表。

* `removedNodes` {NodeList}  
  返回移除的节点，无移除则返回空节点列表。

* `previousSibling` {Node}  
  返回新增/移除的节点的上一个相邻节点，若没有，则为 `null`。

* `nextSibling` {Node}  
  返回新增、移除的节点的下一个相邻节点，若没有，则为 `null`。

* `attributeName` {String}  
  返回被改变的 `attribute` 的本地名称，或 `null`。

* `attributeNamespace` {String}  
  返回被改变的`attribute`的命名空间，或 `null`。

* `oldValue` {String}  
  视 `type` 字段的不同有不同结果：
    - attributes：返回`attribute`被改变前的值
    - characterData：返回改变前的 data
    - childList：返回`null`

## MutationObserver 实例方法

* `observe(target, initOptions)` {void}      
  这个方法为 MutationObserver 实例注册需要观察的节点。  
  参数 `target` 为被观察的节点。  
  参数 `initOptions` 为初始化选项，用以指定需要观察哪些改变(mutation)。  
  可用的选项有：  

  - childList {Boolean}  
    若 `true`，表示需要观察目标节点的子节点（包括文本节点）的增删。

  - attributes {Boolean}  
    若 `true`，表示要观察目标节点的 attributes。

  - characterData {Boolean}
    若 `true`，表示需要观察目标节点的 data。

  - subtree {Boolean}  
    若 `true`，表示需要观察目标节点以及其后代节点。

  - attributeOldValue {Boolean}
    若 `true`，表示需要记录 attribute 改变前的值。
  
  - characterDataOldValue {Boolean}  
    若 `true`，表示需要记录 data 改变前的值。

  - attributeFilter {String[]}
    如果只是部分 attribute 需要观察，可以设置这个属性，只需将这些需要观察的 attribute 的名字（无需命名空间）用数组列出来即可。

  > 注意，`childList`，`attributes`，`characterData` 三种选项中，必须最少有一个为 `true`，否则抛`An invalid or illegal string was specified`错误。


* `disconnect()` {void}
  用以停止观察 DOM 改变。如果要恢复观察，需要再次调用 `observe()` 方法。

* `takeRecords()` {MutationRecord[]}  
  用以清空改变记录，并返回这些被清空的记录。

  

## 使用：

### HTML:

```html
<body>
  <div id="target"></div>
</body>
```

### JavaScript:
```js
let target = document.getElementById('target')

// 创建实例，编写 mutation 处理器
// 监控到每次 mutation，都打印出 mutation 的 type
let observer = new MutationObserver(function(mutations, observer) {
  mutations.forEach(mutation => {
    console.log(mutation.type)
  })
})

// 观察节点变化
observer.observe(target, {
  attributes: true, // 观察 attributes 改变
  childList: true, // 观察节点增删
  characterData: true, // 观察文本改变
  subtree: true // 观察子节点树
})

// 修改attribute（触发 type: attributes）
target.setAttribute('attr', 'attr')
// 插入节点（触发 type: childList）
let child = target.appendChild(document.createTextNode('oldTextValue'))
// 修改文本（触发 type: characterData）
child.nodeValue = 'characterData'

// 停止观察，注意：
// MutationObserver 的回调函数是在 Microtask quene 里的，
// 如果这里不用 setTimeout 将其放入 Macrotask quene 延后到下次事件循环，
// 上面的 DOM 修改将不能生效，因为还没执行本次事件循环的 Microtask，
// observer 就已经被 "disconnect" 了。
setTimeout(()=>{ observer.disconnect() }, 0)

```

### 结果输出：

```
attributes
childList
characterData
```

## 浏览器兼容性

* chrome 18+
* Safari(Mac) 6+
* Safari(iOS) 6.0+
* Android Browser 4.4+
* Firefox 14+
* IE 11
* Opera 15+

不出所料，除了 IE ，主流浏览器都能兼容。
