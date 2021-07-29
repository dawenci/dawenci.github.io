---
layout: post
title:  "JavaScript 函数式编程参数数量问题"
categories: JavaScript
tags: [JavaScript, FP]
---


## map 的参数数量

看一个经典的例子：

```js
['123', '123'].map(parseInt)
// [123, NaN]
```

原因在于，`map` 的回调，会传入三个参数，而 `parseInt` 函数的签名为 `parseInt(string, radix)`，第二个参数 `radix` 为一个 `2` ~ `36` 之间的基数，代表以什么进制去解析第一个参数 `string`，而一旦无法解析，则会返回 `NaN`。这个例子中，`map` 第二个 `123` 时，传入 `parseInt` 的参数为 `('123', 1, ['123', '123'])`，所以返回的结果就出乎意料了。

解决这种问题，有两种思路。

第一种，多一层拦截，限制传入 `parseInt` 的参数。

定义一个辅助函数

```js
function unary(fn) {
  return function(arg) {
    return fn(arg)
  }
}
```

该函数的作用为，包裹一个原始函数，确保只会传入一个参数到该原始函数。

使用：

```js
['123', '123'].map(unary(parseInt))
// [123, 123]
```

这样结果就正确了。

第二种思路，我们定制 `map` 方法，只为回调传入每个元素作为参数，不传入多余的参数。

```js
function map(iteratee, list) {
  if (!list || !list.length) list = []
  const len = list.length || 0
  const results = Array(len)
  for (let index = 0; index < len; index += 1) {
    // 只传入一个参数
    results[index] = iteratee(list[index])
  }
  return results
}
```

使用:

```js
map(parseInt, ['123', '123'])
// [123, 123]
```

这种方法，比较匹配函数式编程风格，被一些函数式编程库采用，例如 `Ramda`。

> 函数式编程库倾向于使用单一参数，这样可以方便于函数组合，更好地实现 “Pointfree” 编程风格。

不过集合操作函数，只为回调传入一个参数，虽然非常方便，但有时候我们真的需要用到下标，而且这种场景还不少，因此需要一种合理的解决方案。

我们当然可以为每个函数写一个双参数的版本，诸如 `mapIndexed`, `forEachIndexed` 等等。例如：


```js
function mapIndexed(iteratee, list) {
  if (!list || !list.length) list = []
  const len = list.length || 0
  const results = Array(len)
  for (let index = 0; index < len; index += 1) {
    // 传入两个参数
    results[index] = iteratee(list[index], index)
  }
  return results
}
```

使用：

```js
mapIndexed((a, b) => `${a}-${b}`, ['a', 'b', 'c'])
// ["a-0", "b-1", "c-2"]
```

这种实现的好处是，代码运行效率非常高，实现也非常简单。

除此之外，还有一种做法，就是采用 `Ramda` 那样的方式，编写一个通用的 `addIndex` 函数，基于原集合操作函数，生成一个带 index 的版本。


```js
const slice = Array.prototype.slice
function addIndex(originFn) {
  let newFn = function(orginIteratee) {
    const args = slice.call(arguments, 0)

    let index = 0

    // 加一层包裹，额外传递 index 进去
    const newIteratee = function(a, b, c) {
      let result
      switch (arguments.length) {
        case 1: {
          result = orginIteratee(a, index)
          break
        }
        case 2: {
          result = orginIteratee(a, b, index)
          break
        }
        case 3: {
          result = orginIteratee(a, b, c, index)
          break
        }
        default: {
          const args = slice.call(arguments, 0)
          args.push(index)
          result = orginIteratee.apply(void 0, args)
        }
      }
      index += 1
      return result
    }

    args[0] = newIteratee
    return originFn.apply(void 0, args)
  }

  return newFn
}

```

```js
const mapIndexed = addIndex(map)
mapIndexed((a, b) => `${a}-${b}`, ['a', 'b', 'c'])
// ["a-0", "b-1", "c-2"]
```

>  注： `Ramda` 的实现，还会额外传入第三个参数，即操作的列表本身，跟 ES 原生的方法一样。

