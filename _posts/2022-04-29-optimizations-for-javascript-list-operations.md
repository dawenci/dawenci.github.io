---
layout: post
title:  "JavaScript 列表数据操作的优化方案"
categories: JavaScript
tags: [JavaScript, Lazy, FP, Sequence]
---

## 从一个简单需求说起

考虑下面需求：

我们有一组日常开支记账列表，我们需要从列表中，过滤出所有咖啡的消费，然后转换成消费金额，取前三条数据。

表达为代码实现，大概是这样的：

```js
const list = [
  { type: '早餐', amount: 12 },
  { type: '咖啡', amount: 25 },
  { type: '午餐', amount: 18 },
  { type: '咖啡', amount: 19 },
  { type: '零食', amount: 8 },
  { type: '晚餐', amount: 22 },
  { type: '水果', amount: 20 },
  { type: '早餐', amount: 12 },
  { type: '咖啡', amount: 25 },
  { type: '午饭', amount: 20 },
  { type: '咖啡', amount: 19 },
  { type: '晚饭', amount: 21 },
  //还有许多...
]

const result = list.filter(item => item.type === '咖啡').map(item => item.amount).slice(0, 3)
```

那么这个实现中，存在哪些比较显而易见的问题呢？

---

## 问题分析

例子代码中起码存在着以下问题：

<!-- more -->

### 没必要的中间临时数组生成

list 执行每一步操作的结果，都会生成一个中间的新数组，例如 filter 的结果，会生成一个 list 的子集数组，  
map 的执行结果，会将这个子集映射为另外一个新数组，  
这些都是中间临时结果，对我们解决问题并没有实际意义。

### 不必要的计算导致性能浪费

上面 list 在执行运算过程中，存在着非必要的计算。  
比如说，我们只需要最终结果的前 3 条数据，但我们的每一步计算方法都完整遍历整个数组，尽忠尽职地完成了所有计算。filter 应用在所有数据上，map 应用在所有 filter 结果上，而最后只切出了一小部分数据而舍弃掉其他同样经过充分计算的数据。这就意味着存在性能的浪费。

所以，我们希望使用使用一种新方案，能改善这些问题。

---

## 问题解决的思路

其实这个问题，在另一篇关于 [transduce](https://dawenci.me/posts/20191213-javascript-transducers) 的文章中，提供了一种可行的解决方案。

而这篇文章，将从另外一个角度来思考和尝试解决。

回到问题，我们想想不使用函数式 API，使用传统的迭代时是怎么做的：

```js
const results = []
for (let i = 0; i < list.length; i += 1) {
  const item = list[i]
  // 相当于 filter
  if (item.type !== '咖啡') continue
  // 相当于 map
  results.push(item.amount)
  // 相当于 slice
  if (results.length === 3) break
}
console.log(results)
```

分析这个迭代方案的程序执行流，可以发现既不存在中间临时数组的生成，也不存在多余的计算。  

从形式上看，迭代的方案，以迭代为视角，一轮迭代中，就整合了数据 filter 过滤、数据 map 映射以及数据 slice 切片提取，每个操作的结果都直接影响下一步操作，无需暂存生成临时数据整个算法都是紧凑高效的。

而前文采用的函数式 API 中，则是以操作为视角，每种操作单独一轮迭代来实现。这种方式，各种算法可以更灵活地组合使用，逻辑组织也更加清晰。但是也因为操作之间比较独立，需要完整保留中间结果用于传递给后续操作，造成性能浪费。

综合这些背景信息，给我们即将设计的解决方案，指引了一个方向。
我们需要实现一种机制，既可以使用函数式 API 来组织我们的代码逻辑，而在内部执行时，又要可以用迭代的方案，将各类操作组合在一起执行高效计算，同时还要有按需计算的特性，需要多少结果就计算多少次杜绝浪费。

要实现这些目标，首先我们的函数 API 就不能那么“勤恳”，一调用就立即完全遍历数据并一股脑执行操作到底，而应当足够 “lazy” ，调用的时候，只是收集我们的整个运算过程的意图，保存我们的计算过程，给我们机会去优化组合整个计算，最后在需要计算结果的时候，再执行计算。

按照上述分析的思路，我们希望代码能写成类似下面的形式：

```js
const results = list
  .filter(item => item.type === '咖啡')
  .map(item => item.amount)
  .take(3)
```

且代码的执行过程是这样的：

1. take 需要 3 条数据，先找上游的 map 运算要一条经过 map 处理的数据
2. map 需要一条数据进行运算，找上游的 filter 运算要一条经过 filter 的数据
3. filter 运算需要一条数据，找上游的 list 取一条数据
4. filter 从上游的 list 拿到一条数据，执行判断，如果不符合要求，则继续找 list 要一个数据，直到测试通过，则将结果答复给下游的 map
5. map 从上游的 filter 拿到一条数据，执行 map 逻辑，将执行结果答复给下游的 take
6. take 从上游的 map 成功拿到一条数据，检测拿够 3 条了没，如果够了，则计算停止，否则继续 1-6 步骤

---

## 问题解决方案的实现

从这个代码执行流程中可以看出，重点是在向上一个环节拿一条数据这个动作，这使用迭代器模式即可完美实现。  
我们在每一步接口调用收集运算意图的时候，可以创建一个对应当前环节运算的可迭代对象，其迭代器实现中，每一次迭代，都访问上游的可迭代对象的迭代器拿一条经过处理的数据，然后应用自己的计算逻辑。同样道理，当前环节的下游计算，也通过其迭代器实现中，通过当前环节提供的迭代器拿数据。

按照这个思路，我们先写一个简单的实现，代码中将使用注释来解释：

```js
// 这是调用 map 操作时，创建的可迭代对象
class MapIterable {
  constructor(iterable, f) {
    // 连接上游的迭代器
    this.iterable = iterable
    // 记录 map 操作
    this.f = f
  }

  // map 的迭代器实现，可以从上游拿数据，应用 f 操作后，向下游提供一条 map 后的数据
  *[Symbol.iterator]() {
    // 调用上游的迭代器，从上游取数据
    for (let v of this.iterable) {
      // 向下游提供 map 操作后的数据
      yield this.f(v)
    }
  }
}

class FilterIterable {
  constructor(iterable, f) {
    // 连接上游的迭代器
    this.iterable = iterable
    // 记录 filter 操作
    this.f = f
  }

  // filter 的迭代器实现，可以从上游拿数据，使用 f 谓语检测，检测通过，则提供给下游
  *[Symbol.iterator]() {
    // 调用上游的迭代器，从上游取数据
    for (let v of this.iterable) {
      // 判断检测结果，符合则向下游提供数据，否则则从上游继续取数据做测试
      if (this.f(v)) yield v
    }
  }
}

class TakeIterable {
  constructor(iterable, n) {
    // 连接上游的迭代器
    this.iterable = iterable
    // 记录需要 take 的数量
    this.n = n
  }

  // take 的迭代器实现，可以从上游拿数据，进行数量检测确定是否计算终止
  *[Symbol.iterator]() {
    let count = 0
    for (let v of this.iterable) {
      if ((count++) < this.n) yield v
      else break
    }
  }
}

// 实现一组对外 API，实现 map、filter、take 等方法，以及一个最终求值的方法 value()
const map = (f, iterable) => new MapIterable(iterable, f)
const filter = (f, iterable) => new FilterIterable(iterable, f)
const take = (n, iterable) => new TakeIterable(iterable, n)
const value = (iterable) => [...iterable]
```

有了这个实现后，回到上面的问题，目前就可以写出以下代码：

```js
const results = value(take(3, map(item => item.amount, filter(item => item.type === '咖啡', list))))
console.log(results)
```

> 注意，数组本身就是实现了 `[Symbol.iterator]` 接口的。相似的，`Set`, `Map` 等内建类型，也都是。

执行结果，完美出现

```js
[25, 19, 25]
```

功能核心实现了，但是这种嵌套调用的代码很反人类，分步骤书写可以稍微改善下：
```js
let seq = list
seq = filter(item => item.type === '咖啡', seq)
seq = map(item => item.amount, seq)
seq = take(3, seq)
const results = value(seq)
console.log(results)
```

但是还是不理想，干扰信息太多。

### 使用函数式风格优化

我们可以使用函数式风格的方案，提供一个 pipe 方法顺序执行一批[柯里化](https://dawenci.me/posts/20190804-currying)后的函数：
```js
const pipe = (fns) => {
  const l = fns.length
  return (x) => {
    let i = -1
    while (++i < l) x = fns[i](x)
    return x
  }
}
const curry1 = (f) => {
  return function curry_f(a) {
    if (arguments.length > 0) return f(a)
    return curry_f
  }
}
const curry2 = (f) => {
  return function curry_f(a, b) {
    if (arguments.length > 1) return f(a, b)
    if (arguments.length === 1)
      return curry1(function (b) {
        return f(a, b)
      })
    return curry_f
  }
}
```

然后重构下我们的 api 以支持自动柯里化：
```js
const map = curry2((f, iterable) => new MapIterable(iterable, f))
const filter = curry2((f, iterable) => new FilterIterable(iterable, f))
const take = curry2((n, iterable) => new TakeIterable(iterable, n))
const value = curry1((iterable) => [...iterable])
```

接下来就可以愉快地使用下面的风格了：

```js
// applyActions 调用将按照顺序执行 filter, map, take, value 4 个操作，
// 每个操作的入参都是前一个操作的结果输出，第一个操作的参数从外部传入
const applyActions = pipe([
  filter(item => item.type === "咖啡"),
  map(item => item.amount),
  take(3),
  value,
])
const results = applyActions(list)
console.log(results)
```

如此一来，可读性显著提高，逻辑清晰了许多。


### 链式调用

上文函数式风格的代码，并不能取悦所有的人。比如有些用户更偏好链式操作，该怎么办？

当然也没问题，我们可以提供一个通用的对外的类型，这个类型可以持有我们实现的所有的 Iterable 对象，并且在这个类型上挂载所有的操作方法。直接代码实现示范：


```js
// 对外类型暴露的类型，用于包含可迭代对象
class Sequence {
  constructor(iterable) {
    // 持有一个可迭代对象
    this.iterable = iterable
  }
  // 这里未实现操作方法，为了方便扩展，我们直接在外部提供方法用于扩展当前类的原型
}

// 提供一个方法定义工具函数
const defineMethod = (name, impl) => {
  Sequence.prototype[name] = impl
}

// 往我们对外暴露的 Sequence 类上添加我们需要的方法
// 每个需要链式调用的 api，都返回一个新的 Sequence 对象
defineMethod('filter', function (f) {
  return new Sequence(filter(f, this.iterable))
})
defineMethod('map', function (f) {
  return new Sequence(map(f, this.iterable))
})
defineMethod('take', function (n) {
  return new Sequence(take(n, this.iterable))
})
defineMethod('value', function (f) {
  return value(this.iterable)
})
defineMethod('find', function (f) {
  return find(f, this.iterable)
})
```

之后就可以使用链式调用 API 啦：

```js
const results = new Sequence(list)
  .filter(item => item.type === '咖啡')
  .map(item => item.amount)
  .take(3)
  .value()
console.log(results)
```

---


## 扩展，支持更多操作

通过这个实现例子，我们可以依葫芦画瓢，支持更多的算法，都不是问题。
从形式上来看，可以新增的 api 主要有两大类型。

一类是类似上文的 `map`、`filter`、`take` 这样的，仍然返回一个包含运算逻辑的可迭代对象，这种可以可以串起来调用以实现更复杂的运算。例如可以实现一个 `drop` 函数，返回一个 `DropIterable` 类型的对象，用来移除列表头部的若干元素，具体实现代码限于篇幅不再展开演示。

另一类是返回非可迭代的对象，用于获取一些其他结果，例如，我们演示如何新增一个 find 函数：

```js
const find = curry2((f, iterable) => {
  for (let v of iterable) {
    if (f(v)) return v
  }
})
defineMethod('find', function (f) {
  return find(f, this.iterable)
})

const result = new Sequence(list)
  .filter(item => item.type === '咖啡')
  .find(item => item.amount < 20)
console.log(result)
```

打印结果:
```js
{type: '咖啡', amount: 19}
```

---


## 扩展，支持更多数据类型

我们目前所实现的所有操作，都有个前提，就是源头数据必须是可迭代的。  
所以，支持更多的数据源类型，本质上就是为这种数据类型实现一个迭代器。  
比如说，我们要支持类数组对象（具有 `lenth`，可以通过 `data[n]` 形式使用下标访问数据）作为数据源，就可以写一个数据包装类，在这个类中提供迭代器以访问对象的数据：

```js
class ArrayLikeIterable {
  constructor(arrayLike) {
    this.arrayLike = arrayLike
  }

  *[Symbol.iterator]() {
    const len = this.arrayLike.length
    for (let i = 0; i < len; i += 1) {
      yield this.arrayLike[i]
    }
  }
}
```

测试下：

```js
const arrayLike = {
  length: 4,
  0: 0,
  1: 1,
  2: 2,
  3: 3,
}

const results = new Sequence(new ArrayLikeIterable(arrayLike))
  .filter(n => n % 2)
  .map(n => n + 2)
  .value()
console.log(results)
```
结果正确打印出

```js
[3, 5]
```

### 无限列表

除了可以对常规的数据结构提供支持，基于迭代器模式带来的能力，我们还可以很方便地支持无限元素的列表结构：

```js
class InfinityIterable {
  constructor(f) {
    this.f = f
  }
  *[Symbol.iterator]() {
    let i = 0
    while (true) yield this.f(i++)
  }
}

const results = new Sequence(new InfinityIterable(x => x))
  .filter(n => n % 2)
  .take(10)
  .value()
console.log(results)
```

结果打印出：
```js
[1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
```

> 注，无限列表的使用，不能简单应用 filter、map 等操作，因为会死循环。所以终究需要配合上文提到的 take、find 等函数，让计算终止。

---

本文相关代码，可在 [github](https://github.com/dawenci/sequence) 查看更完整的版本。

---

全文完
