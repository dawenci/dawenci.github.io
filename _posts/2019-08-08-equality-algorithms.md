---
layout: post
title:  "JavaScript 中的 “相等” 算法"
categories: JavaScript
tags: [JavaScript]
---

整理笔记本时，看到以往一些对 JavaScript 中 “相等” 关系的记录。
尽管这是一个被说烂了的话题，但写一篇资料整理总结，当作复习下也不错。

JavaScript 的 “相等” 比较算法，按严格程度来排，有以下几种：

1. `==` 双等号
2. `===` 三等号
3. `SameValueZero` 算法
4. `SameValue` 算法

下面逐个介绍它们的特点。

<!-- more -->

---

## `==` 双等号

在 JavaScript 中，使用 `==` 来判断相等，是一种不严格的比较。

关于双等号，最广为人知的一点，就是在比较不同类型的值时，JavaScript 会进行自动转型。

如果不明白其转型规则，可能会出现让人匪夷所思的结果。

举例来说：

```js
'' == false // true
'  ' == false // true
'' == ' ' // false
```

具体的转型规则如下：

1. 两边是对象（引用类型），比较内存地址是否相同。
2. 一边是原始类型，一边是对象，则将对象转换成原始类型再比较
3. 两边都是原始类型：
  - 3.1. 一边是 Symbol，则返回 `false`
  - 3.2. 两边类型一样，直接比较值
  - 3.3. 一边为 `undefined` 或 `null`，则比较另一边是否也为 `undefined` 或 `null`
  - 3.4. 一边为 “数字” 或者 “布尔值”，则以 “数字” 进行比较


看下面代码理解：

```javascript

// 结果 false，两边都是原始类型，类型相同，非 Symbol，直接比较值
'' == '   '

// 结果 true，两边都是原始类型，一边是布尔值，则以数字进行比较，两边都是 `0`
false == '' 

// 结果 true，两边都是原始类型，一边是布尔值，则以数字进行比较，两边都是 `0`
false == '    '

// 结果 true。
// 数组先转原始类型，尝试 valueOf 返回数组不采纳，
// 继续 toString 返回空字符串，采纳，
// 然后一边为布尔值，则以数字进行比较，
// 空字符串和 false 都转换成 0，于是判断成立
[] == false

// 结果 true。
// 数组先转原始类型，尝试 valueOf 返回数组不采纳，
// 继续 toString 返回空字符串，采纳，
// 然后一边为数字，则以数字进行比较，
// 空字符串转换成 0，于是判断成立
[] == 0

// 结果 true。
// 对象先转原始类型，尝试 valueOf，获得原始类型值，进行比较
({ valueOf: function() { return false } }) == false
({ valueOf: function() { return '' } }) == false
({ valueOf: function() { return 0 } }) == false
({ valueOf: function() { return 1 } }) == true
({ valueOf: function() { return 2 } }) != true
({ valueOf: function() { return 2 } }) != false
```

由此可见，使用 `==` 进行比较，许多情况下，结果可以说是非常诡异的，稍不注意就可能出错。而且记忆这些转换规则的负担也很大。
所以，通常我们在制定代码规范的时候，都会增加一条总是使用 `===` 的规则来绕过避开这种复杂性。

尽管如此，双等号用于判断 `null` 和 `undefined` 的时候，还是比较简洁的：

```js
// 使用 == null 可以同时处理 null 和 undefined 两种情况
if (name == null) {
  // ...
}

// 等价于
if (name === null || name === undefined) {
  // ...
}

```

所以，如果真要用，个人推荐仅用于这种判断 `null` 的情况。

---

## `===` 三等号

使用三个 `===`，可以强制让 JavaScript 不自动转型，进行严格比较：

```js
'1' === 1 // false
```

从而避开了上述 `==` 的一系列坑点。

但是，三等号比较也不总是完全符合人们的直觉，例如考虑下面的情况：

```js
NaN === NaN // false
```

还有：

```js
-0 === 0 // true

// 但：
;(1 / -0) === (1 / 0) // false
```

所以，我们还需要其他比较算法来应付这种情况。

---

## SameValueZero

SameValueZero 算法，其在 `===` 的基础上，增加了对 `NaN` 的特殊处理，`NaN` 和 `NaN` 之间在该算法中，是相等的。
该算法在语言中有众多应用，比如语言内置的 `TypedArray`（如 `Float64Array` 等等）、`ArrayBuffer`、`Map`、`Set` 等等。例如：

```js
const set = new Set([NaN, NaN]) // Set(1) {NaN}
console.log(set.size) // 1
set.has(NaN) // true

const map = new Map([[NaN, NaN], [NaN, NaN]]) // Map(1) {NaN => NaN}
console.log(map.size) // 1
map.has(NaN) // true

const arr = new Float64Array([NaN]) // Float64Array [NaN]
arr.includes(NaN) // true
```

值得关注的是，ES7 中的 `Array.prototype.includes` 也使用了该比较算法，如下：

```js
// 旧 ES 版本检查数组是否存在元素经常使用 indexOf，但无法处理 NaN
[NaN].indexOf(NaN) !== -1 // false

// 而新版本的方案，使用 includes，可以完美支持 NaN
[NaN].includes(NaN) // true
```

了解了 SameValueZero 算法的特性后，我们就可以简单地写出实现了：

```js
function sameValueZero(a, b) {
  if (a === b || (a !== a && b !== b)) return true
  return false
}
```

核心是利用了 `NaN !== NaN` 的奇葩性质，来特殊处理两个 `NaN` 的比较。

---

## SameValue

以上的三种相等算法，都无法区分正负 `0`，SameValue 算法的行为，正是在 SameValueZero 的基础上, 再增加对正负 `0` 的区分处理。

JavaScript 中，内置有 SameValue 算法的实现：`Object.is`：

```js
Object.is(0, 0) // true
Object.is(0, -0) // false
Object.is(NaN, NaN) // true
```

当然，自己实现也并不麻烦：

```js
const sameValue = typeof Object.is === 'function'
  ? Object.is
  : function(a, b) {
    if (typeof a === 'number') {
      // +0, -0 的情况
      if (a === b) return 1 / a === 1 / b
      // a，b 均为 NaN 返回 true
      if (a !== a && b !== b) return true
      return false
    }
    return a === b
  }
```

实现的核心就是利用正数除以 `+0` 和 `-0`，
会得到 `Infinity` 和 `-Infinity` 的性质，对 `+-0` 进行区分。

---

全文完
