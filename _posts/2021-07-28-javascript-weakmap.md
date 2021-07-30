---
layout: post
title:  "JavaScript WeakMap"
categories: JavaScript
tags: [JavaScript, WeakMap, polyfill]
---

## 语言实现

比较无聊，不谈


## 怎么 polyfill

分析：

1. 不能阻止 key 被 gc，因此，WeakMap 中不能直接持有 key。
2. 通过 key 能找到 value，结合上条，只能在 key 中用某种方式去引用 value。
3. 基于上条，key 必须为对象。
4. 一个 key 在多个 WeakMap 中，能引用不同的 value，因此，key 必须能识别不同的 WeakMap，因此，WeakMap 实现中，必须有个唯一标识。
5. 由于 WeakMap 不能持有 key，因此，也无法实现遍历 key、value 等操作。只能实现 get, set, has, delete 等方法。



基于这些分析，

我们可以这样来实现：

1. 每个 WeakMap 构造时，生成一个唯一 id（Symbol 防冲突）。
2. key 中，以上述唯一 id 作为字段名，引用 value。

于是一个可行的实现的核心代码如下：


```js
class WeakMap {
  _uid = Symbol()

  constructor(entries) {
    // 略
  }

  get(key) {
    return key[this._uid]
  }

  set(key, value) {
    Object.defineProperty(key, this._uid, {
      value: [key, value],
      configurable: true,
      enumerable: false,
      writable: true
    })
  }

  delete(key) {
    delete key[this._uid]
  }

  has(key) {
    return key.hasOwnProperty(this._uid)
  }
}

```
