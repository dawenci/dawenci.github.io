---
layout: post
title:  "arguments 对象与形参联动关系"
categories: JavaScript
tags: [JavaScript, Arguments, ES5]
---

strict mode 中，函数的 `arguments` 对象的属性值，就是传入该函数的实参的简单拷贝，它们与形参之间的值**不存在动态的联动关系**。

非严格模式下的函数，函数的 `arguments` 对象的属性与绑定在该函数执行环境中的形参共享值。
换句话说，改变该 arguments 对象的属性将改变对应的形参值，反之亦然。
不过，如果其中一个属性被删除然后再重设值，则会打破这种联动关系。

下面用一个简单的例子来说明这些现象。


## 代码

```js
// strict mode
;(function test(a) {
  'use strict';

  console.log(arguments[0]) // 1
  console.log(a) // 1

  a = 2
  console.log(arguments[0]) // 1
  console.log(a) // 2
})(1)

// 非strict mode
;(function test2(a) {
  // 初始一一对应
  console.log(arguments[0]) // 1
  console.log(a) // 1

  // 修改形参，参数对象联动变化
  a = 2
  arguments[1] = 2
  console.log(arguments[0]) // 2
  console.log(a) // 2

  // delete 属性再重新设值，则不在联动
  delete arguments[0]
  arguments[0] = 3
  console.log(arguments[0]) // 3
  console.log(a) // 2

})(1)

```
