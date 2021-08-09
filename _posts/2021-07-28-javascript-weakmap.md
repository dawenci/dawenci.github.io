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
2. 通过 key 能找到 value，因为上条，不能通过第三方维护这种对应关系，于是只能在 key 中用某种方式去引用 value。
3. 基于上条，key 必须为对象。
4. 同一个对象能作为多个 WeakMap 的 key，分别引用不同的 value，这意味着 key 必须能识别不同的 WeakMap，因此，WeakMap 实现中，必须有个唯一标识，key 通过该标志引用不同的 value。
5. 由于 WeakMap 不负责存储 key、value，所以 WeakMap 无法实现遍历 key、value 等操作。只能实现 get, set, has, delete 等方法去操作 key 中存储的数据。

<!-- more -->

基于这些分析，

我们可以这样来实现：

1. 每个 WeakMap 构造时，生成一个唯一 id（Symbol 防冲突）。
2. key 中，以上述 Symbol 作为属性名，引用 value。

于是一个可行的实现的核心代码就顺理成章出来了：


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

---

全文完
