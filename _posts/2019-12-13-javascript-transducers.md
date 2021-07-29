---
layout: post
title:  "JavaScript 实现 Transducers"
categories: JavaScript
tags: [JavaScript, Transducers, FP]
---

其实第一次听说 transducer 的概念，是在某个周末时间学习 Clojure 的时候。
一开始觉得概念很复杂，因此就没有深入了解，内部的实现机制并不清楚。

直到后来尝试写一个 JavaScript 的函数式编程的函数库时，才做了一番功课去研究。

本文就是记录我对学习过程的思考和结果。

我们不讲概念，而是从一个简单的问题开始。

## 问题

假设你现在在检查上个月的开支清单，里面的条目大概如下：

```js
const list = [
  { type: '早餐', amount: 12 },
  { type: '午餐', amount: 18 },
  { type: '咖啡', amount: 19 },
  { type: '零食', amount: 8 },
  { type: '晚餐', amount: 22 },
  { type: '水果', amount: 20 },
  { type: '早餐', amount: 12 },
  { type: '午饭', amount: 20 },
  { type: '咖啡', amount: 24 },
  { type: '晚饭', amount: 21 },
]
```

如果你现在想知道喝了多少钱的咖啡，按照以往的经验，我们可能会这么做：

```js
const filterCoffee = item => item.type === '咖啡'
const getAmount = item => item.amount
const add = (a, b) => a + b

list
  .filter(filterCoffee)
  .map(getAmount)
  .reduce(add, 0)

// 结果为 43
```

这是一个非常典型的数据处理过程，我们现在来分析，这个过程发生了什么。

1. 列表处理：过滤数据

```js
.filter(filterCoffee)

// 结果为
// [
//   { type: '咖啡', amount: 19 },
//   { type: '咖啡', amount: 24 },
// ]
```

* 总共迭代了 10 个条目，逐个检查 `type`
* **创建了一个新列表**，存放滤出的 2 个符合条件的条目。


2. 列表处理：映射数据

```js
map(getAmount)
// 结果为
// [
//   19,
//   24,
// ]
```

* 对 `filter` 的结果，进行了迭代处理，总共处理了 2 个条目
* **创建了一个新列表**，存放得到的 2 个金额数值。

3. 执行计算：求和

```js
reduce(add, 0)
// 结果为 43
```

* 对 `map` 的结果，进行了累加，总共处理了两个条目
* 得到一个数值 43


以上过程，我们可以总结出一些特点：
1. 对列表的每项操作都是独立连续完成的，做完一项，再进行下一项。
2. 每一次操作，都**创建自己的结果存储（中间数据）**，传递给下一个操作作为输入。

这中间存在着什么问题？

中间数据算一个问题，我们并不关心计算的中间过程，这些中间生成的过程数据，只是我们使用这种流式的实现方式不得已产生的。
那么，能否消除这些中间数据的创建呢？使用本文的主题，transducer 即可改善这一点。

但是，为了讲清楚，暂时先不展开，我们还需要理解一些铺垫的概念。

### reduce

让我们重新审视下 reduce 函数。

reduce 可能是我们常用的列表处理方法中，最复杂的一个了。
它的回调除了接受每个列表元素，还有个特别的“累积值”，而且除了回调，还接受一个初始值（非必须）。它可以：

1. 可以对列表做完整的遍历，访问到所有元素（current）及其 index
2. 提供（累积值）acc 作为操作结果的存取变量

实际上，reduce 这么复杂，是因为，它其实是将对列表的递归操作模版化了。所以，通过递归或者迭代列表能做到的事情，用 reduce 一样能做。

有了这个基础之后，我们就可以基于 reduce 来做各种各样的列表操作。

看一个例子：

```js
list.map(item => item.type)
// ["早餐", "午餐", "咖啡", "零食", "晚餐", "水果", "早餐", "午饭", "咖啡", "晚饭"]
```

我们如果使用 reduce 来实现同样的行为，可以这么做：

```js
list.reduce((acc, item) => (acc.push(item.type), acc), [])
// ["早餐", "午餐", "咖啡", "零食", "晚餐", "水果", "早餐", "午饭", "咖啡", "晚饭"]
```

上面两种写法，它们表达的含义是一样的：
1. 将数据做变换（ 提取 `type` ）
2. 返回所有数据变换后的结果列表

结论是，reduce 可以做到 map 能做到的事情。

再看一个例子：

```js
list.filter(item => item.type === '咖啡')
// [{type: "咖啡", amount: 19}, {type: "咖啡", amount: 24}]
```

改用 reduce 实现：

```js
list.reduce((acc, item) => (item.type === '咖啡' ? (acc.push(item), acc) : acc), [])
```

同样，这两段代码也是一样的含义
1. 使用谓语 ( type 为“咖啡” ) 对数据做过滤
2. 返回所有过滤的结果列表

同样可以得出结论，reduce 可以做到 filter 能做到的事情。

综合上面两个例子，我们可以找出一个共同点，如果使用 reduce 来做列表操作，所有功能实现的核心，都是由两个子部分组成的：

1. 数据加工
2. 结果处理

对于 map 来说，数据加工，就是将数据的映射方法，作用到每个元素上；
对于 filter 来说，就是使用谓语来检查数据是否需要累积。
而结果处理，则是将加工后的结果添加入结果列表中。

扩展一下，通过同样的思路，我们可以将所有常用的列表操作，都改成使用 reduce 来做。

上面铺垫这么多，就是为了得出一个结论，我们可以借助 reduce ，使用统一的方式来做列表操作，这种统一，是下一步实现的前提。

现在我们先回到开头的例子，我们统一使用 reduce 来做列表操作，可以将代码改成：

```js
const filterCoffeeReducer = (acc, item) => (item.type === '咖啡' ? (acc.push(item), acc) : acc)
const mapAmountReducer = (acc, item) => (acc.push(item.amount), acc)
const sumReducer = (acc, amount) => (acc + amount)

list
  .reduce(filterCoffeeReducer, [])
  .reduce(mapAmountReducer, [])
  .reduce(sumReducer, 0)
```

### transducer

上面使用流式的函数处理列表过程，每一步变换，其实可以抽取出来。
在数据变换环节中，组合这些变换操作成一个单独的变换操作，最后将这个组合而成的变换加工出来的数据，累积到结果上，从而一步到位，无需生成中间列表数据就获得最终结果。
这就是 tansducer 做的事情。

那么要如何来抽取组合变换逻辑呢？




我们手工做一次改写，将 filterCoffeeReducer 和 mapAmountReducer 部分组合起来：

```js
list
  .reduce((acc, item) => (item.type === '咖啡' ? (acc.push(item), acc) : acc), [])
  .reduce((acc, item) => (acc.push(item.amount), acc), [])
  .reduce((acc, amount) => (acc + amount), 0)
```


如果使用 transducer 来解决问题，伪代码描述，大概是这样的：

```
result = list.transduce(
  Transformer,
  数据累积方法,
  列表数据
)
```

其中，Transformer，即变换操作的方法，描述如何将一组操作用于变换原始数据到可以被最后累积处理数据。
对应上面的例子，就是 filter 和 map 的逻辑部分：

1. 检查是否 “咖啡”
2. 转换成金额

Transformer 将这两个操作应用于每个原始数据，直接一步到位，得到可以最后用于计算的金额数据。

数据累积方法，则是最后如何 “累积” 上一环节转换的结果。
对应上面的例子，就是 reduce 部分的逻辑：

1. 累加金额




## 怎么实现

可以看我 github 中的一个简单的 [transduce](https://github.com/dawenci/ijs/blob/master/src/transduce.ts) 实现。

（由于比较复杂，暂时不展开解释，有兴趣就看代码跟注释吧…）

下面贴一些核心的代码：

```typescript

// 协议
const INIT = '@@transducer/init'
const RESULT = '@@transducer/result'
const STEP = '@@transducer/step'
const REDUCED = '@@transducer/reduced'
const VALUE = '@@transducer/value'
const ITERATOR = '@@iterator'

// reducer，即每一步执行 reducing 的函数
interface Reducer {
  (accumulator, currentValue): any
}

interface Reduced<T> {
  [VALUE]: T
  [REDUCED]: boolean
}

interface Transformer {
  
  // 返回初始化值
  [INIT]: () => any
  // 返回最终结果
  [RESULT]: (accumulator: any) => any
  // reducer
  [STEP]: (accumulator: any, currentValue: any) => any
}

interface Transducer {
  (tarnsformer: Transformer): Transformer
}

interface _Iterator {
  [ITERATOR]: (value: any) => any
}

const ITER_SYMBOL = typeof Symbol !== 'undefined' ? Symbol.iterator : ITERATOR

// Transformer 的基类
class BaseTransformer implements Transformer {
  [INIT]() {
    throw new Error('init not implemented')
  }
  [STEP](accumulator, currentValue) {
    throw new Error('step not implemented')
  }
  [RESULT](accumulator) {
    return accumulator
  }
}

function _reduced<T>(obj: T | Reduced<T>): Reduced<T> {
  if (obj && obj[REDUCED]) {
    return obj as Reduced<T>
  }
  return { [VALUE]: obj as T, [REDUCED]: true }
}

// 将普通 reducer 包装成 transformer
class XfWrap extends BaseTranducer {
  constructor(private reducer: Reducer) {
    super()
  }
  [STEP](accumulator, currentValue) {
    return this.reducer(accumulator, currentValue)
  }
}

class XMap extends BaseTransformer {
  constructor(private fn, private transformer) {
    super()
  }
  [STEP](accumulator, currentValue) {
    return this.transformer[STEP](accumulator, this.fn(currentValue))
  }
}

class XFilter extends BaseTransformer {
  constructor(private predicate, private transformer) {
    super()
  }
  [STEP](accumulator, currentValue) {
    return this.predicate(currentValue) ? this.transformer[STEP](accumulator, currentValue) : accumulator
  }
}

function _isArrayLike(test: any): boolean {
  if (Array.isArray(test)) return true
  if (!test || typeof test.length !== 'number' || typeof test === 'string') return false
  if (test.length === 0 && typeof test !== 'function') return true
  if (test.length > 0) return test.hasOwnProperty(0) && test.hasOwnProperty(test.length - 1)
  return false
}

// curry 见上一篇的实现
const mappingTransducer = curry(function(
  fn: (value: any) => any, transformer: Transformer
): Transformer {
  return new XMap(fn, transformer)
})

const filterTransducer = curry(function transducer(
  pred: (value: any) => boolean,
  transformer: Transformer
): Transformer {
  return new XFilter(pred, transformer)
})

// 包装 reducing function 为 Transformer
function wrap(reducer: Reducer | Transformer): Transformer {
  return typeof reducer === 'function'
    ? new XfWrap(reducer)
    : reducer
}

function _arrayReduce(tarnsformer, initialValue, iterable) {
  const size = iterable.length
  const stepFn = tarnsformer[STEP].bind(tarnsformer)
  let accumulator = initialValue
  for (let index = 0; index < size; index += 1) {
    const current = iterable[index]
    accumulator = stepFn(accumulator, current)
    if (accumulator && accumulator[REDUCED]) {
      accumulator = accumulator[VALUE]
      break
    }
  }

  return tarnsformer[RESULT](accumulator)
}

function _iterableReduce(tarnsformer, initialValue, iterable) {
  let step = iterable.next()
  const stepFn = tarnsformer[STEP].bind(tarnsformer)
  let accumulator = initialValue
  while (!step.done) {
    accumulator = stepFn(accumulator, step.value)
    if (accumulator && accumulator[REDUCED]) {
      accumulator = accumulator[VALUE]
      break
    }
    step = iterable.next()
  }

  return tarnsformer[RESULT](accumulator)
}

function _methodReduce(tarnsformer, initialValue, object, methodName) {
  let accumulator = initialValue
  return tarnsformer[RESULT](
    object[methodName](tarnsformer[STEP].bind(tarnsformer), accumulator)
  )
}

export default function _reduce(
  // Reducer | Transformer
  tarnsformer,
  initialValue,
  // Array<any> | ArrayLike<any> | Iterable<any> | Iterator<any> | _Iterator
  iterable
) {
  if (typeof tarnsformer === 'function') {
    tarnsformer = wrap(tarnsformer)
  }

  if (_isArrayLike(iterable)) {
    return _arrayReduce(tarnsformer, initialValue, iterable)
  }

  if (iterable[ITER_SYMBOL] != null) {
    const iterator = iterable[ITER_SYMBOL]()
    return _iterableReduce(tarnsformer, initialValue, iterator)
  }

  if (typeof (iterable as any).next === 'function') {
    const iterator = iterable
    return _iterableReduce(tarnsformer, initialValue, iterator)
  }

  if (typeof (iterable as any).reduce === 'function') {
    return _methodReduce(tarnsformer, initialValue, iterable, 'reduce')
  }

  throw new TypeError('reduce: list must be array or iterable')
}

function transduce(transducer, reducer, initialValue, iterable) {
  const transformer = wrap(reducer)
  return _reduce(transducer(transformer), initialValue, iterable)
}
```

### 协议，实现更多 transducer 例子

实际上，关于在 JavaScript 中实现 transduce，已经有一个主流遵守的 [协议](https://github.com/cognitect-labs/transducers-js#transformer-protocol) 了，感兴趣可以看看一些典型的实现，例如 Ramda.js 的相关代码。



全文完（待有空会补充更多细节，展开阐述）
