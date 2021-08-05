---
layout: post
title:  "如何实现一个符合 Promise/A+ 规范的 Promise"
categories: JavaScript
tags: [JavaScript, Promise]
---

## Promise/A+

要自己实现 Promise，就绕不开 [Promise/A+](https://github.com/promises-aplus/promises-spec) 规范，业界主流的 Promise 实现（包括浏览器实现）均实现了该规范。

Promise/A+ 本身规定的内容比较少，这也是我们实现一个简单的 Promise 可以依据的最简单标准了。

首先，解释一些 Promise/A+ 中的术语概念：

- *promise*  
  promise 是一种行为符合本标准的对象或函数。
- *thenable*  
 thenable 是定义了 `then` 方法的对象或函数。
- *value*  
  value 是任何合法的 JavaScript 值（包含 `undefined`、thenable 或 promise）。
- *exception*  
  exception 是使用 `throw` 语句抛出的值。
- *reason*  
  reason 是一个描述 promise 因何而 rejected 的值。

基于这些基础属于概念，下面开始解读规范的一些核心要点。

<!-- more -->

### promise 的状态

promise 一共有三种状态，分别为 `pending`、`fulfilled`、`rejected`，根据所处状态，有对应的规则：

- 当 promise 处于 `pending` 状态，可以转换为 `fulfilled` 或 `rejected`。
- 当 promise 处于 `fulfilled` 状态，状态就不能再变化了，并且必须拥有一个不可变的 value（指对象引用地址不变，对象的属性还是允许改变的）。
- 当 promise 处于 `rejected` 状态，状态也不能变化了，并且必须拥有一个不可变的 reason（不可变限制同上）。


### then 方法

规范要求 promise 必须提供 `then` 方法。

因为 `then` 是规范中用来访问 promise 的结果值或者拒绝的原因的唯一途径。

`then` 方法接受两个可选的参数：

```js
promise.then(onFulfilled, onRejected)
```

关于 `onFulfilled`:
- 如果不是一个函数，必须被忽略（忽略的意思是，将结果值直接往后面传递，不做处理）。
- 如果是一个函数，那么将在 promise fulfilled 之后调用，调用的参数为 promise 的结果值。注意需要确保 `onFulfilled` 不会在 promise fulfilled 之前被调用，且需要确保不会被调用一次以上。

关于 `onRejected`:
- 如果不是一个函数，必须被忽略（忽略的意思是，将 reason 继续往后面抛，不做处理）
- 如果是一个函数，那么将在 promise rejected 之后调用，调用的参数为 promise 的 reason。注意需要确保 `onRejected` 不会再 promise rejected 之前被调用，且需要确保不会被调用一次以上。

以上两个回调，都要作为函数调用（没有 `this` 值），并且在执行上下文栈中仅包含平台代码之前不能被调用，在实践上，这要求 `onFulfilled` 和 `onRejected` 异步执行。浏览器的 Promise 实现中，它们是作为“微任务”执行的，但是规范中并不做这个要求，也可以实现为事件循环中的“任务”，例如使用 `setTimeout` 或 `setImmediate` 来实现。如果要使用“微任务”，可以考虑使用 `MutationObserver` 或 `process.nextTick` 来实现。

> 关于事件循环以及任务、微任务相关内容，可以阅读我的另一篇博文：[浏览器的事件循环机制](https://dawenci.me/posts/20180218-browser-event-loop)  
> 关于 MutationObserver，可以阅读：[使用 MutationObserver 监视 DOM 变化](https://dawenci.me/posts/20170728-mutation-observer)  

以上就是通过 `then` 访问 value 或 reason 的方式，`then` 的调用，本身会返回一个新的 promise。

注意 `then` 可以被多次调用，多次调用将返回独立的 promise 实例。当 promise fulfilled 或 rejected 时，多个 `onFulfilled` 或 `onRejected` 回调将按照它们所属的 `then` 调用的顺序来执行。

对于 `then` 返回的 promise 的状态转换规则，有几条规则需要了解，以下面的例子说明：

```js
promise2 = promise1.then(onFulfilled, onRejected)
```

- 如果 `onFulfilled` 或 `onRejected` 返回了一个 value `x`，此时将执行 promise2 的决议过程（Resolution Procedure）逻辑，后面详述。
- 如果 `onFulfilled` 或 `onRejected` 抛出了一个异常 `e`，那么 promise2 必须以 `e` 作为 reason 被 rejected。
- 如果 `onFulfilled` 不是一个函数，并且 promise1 已经 fulfilled，那么 promise2 必须以 promise1 相同的 value 被 fulfilled。
- 如果 `onRejected` 不是一个函数，并且 promise1 已经 rejected，那么 promise2 必须以 promise1 相同的 reason 被 rejected。

### promise 的决议过程（Resolution Procedure）

这部分是 promise 的核心也是难点部分。

下面以 `[[Resolve]](promise, x)` 这个抽象操作表示以 `x` 作为 value 决议 `promise` 的过程。

那么 `[[Resolve]](promise, x)` 的步骤过程如下：

1. 如果 `promise` 和 `x` 是同一个对象引用，那么直接抛 `TypeError` 错误（避免循环）。  
2. 如果 `x` 为一个 promise，那么根据其状态：  
  2.1 如果 `x` 状态为 `pending`，`promise` 也必须维持 `pending` 状态直至 `x` 变为 `fulfilled` 或 `rejected`。  
  2.2 如果 `x` 状态为 `fulfilled`，`promise` 也 fulfill 为同样的 value。  
  2.3 如果 `x` 状态为 `rejected`，`promise` 也 reject 为同样的 reason。  
3. 如果 `x` 为一个对象（不含 null）或函数，则：  
  3.1. 令 `then` 为 `x.then`。  
  3.2. 如读取属性 `x.then` 导致抛出异常 `e`，则以 `e` 作为原因 reject 掉 `promise`。  
  3.3. 如 `then` 为一个函数，则使用 `x` 作为 `this` 调用 `then`，并传入两个回调函数 `resolvePromise` 及 `rejectPromise`，对于这两个参数：  
    3.3.1. 当传入 value `y` 调用 `resolvePromise(y)` 时，执行 `[[Resolve]](promise, y)`。  
    3.3.2. 当传入 reason `r` 调用 `rejectPromise(r)` 时，以 `r` 作为 reason reject `promise`。  
    3.3.3. 当 `resolvePromise` 或 `rejectPromise` 两者合起来被调用超过一次，则除了第一个调用，其余均会被忽略。  
    3.3.4. 当调用 `then` 时抛出一个异常 `e`，则：  
      3.3.4.1 如果已经调用过 `resolvePromise` 或 `rejectPromise`，直接忽略异常。  
      3.3.4.2 否则以 `e` 作为 reason reject `promise`。  
  3.4. 如果 `then` 不是函数，则使用 value `x` 来 fulfilled `promise`。  
4. 如果 `x` 既非对象也非函数，使用 value `x` 来了 fulfilled `promise`。

> 如果一个 promise 以 thenable 为 value 被 resolved，即 `[[Resolve]](promise, thenable)`，并且这个 thenable 是一个循环的 thenable 链的一环，那么将会导致 `[[Resolve]](promise, thenable)` 再次被调用，出现无尽递归。规范虽然不要求，但是鼓励检测这样的循环，并以包含详细信息的 `TypeError` 作为 reason 来 reject 这个 promise。  
> 上面对 thenable 的处理方式，使得不同的 Promise/A+ 实现可以互操作。


以上就是对 Promise/A+ 规范的要点解读，虽然内容不多，但是也非常枯燥无聊，下面开始一步步写一个简单的实现。

---

## 实现

仿照浏览器的 Promise 实现，我们也准备将自定义的实现以 `class` 的形式来实现，先按照上述规范要求，写出最主要的结构：

```js
class PromiseAplus {
  then(onFulfilled, onRejected) {}
}
```

跟着，补充完整基础的属性，包括状态、value 和 reason：

```js
// 定义 Promise 的三种状态
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'
const PENDING = 'pending'

class PromiseAplus {
  constructor() {
    // promise 刚创建时，为 PENDING 状态
    this._PromiseState = PENDING

    // 由于 `fulfilled` 和 `rejected` 永远互斥，
    // 因此，value 跟 reason 我们统一使用一个属性保存即可
    // promise 创建时，value 初始化为 undefined
    this._PromiseResult = undefined
  }

  then(onFulfilled, onRejected) {}
}
```

promise 存在从 `pending` 往 `fulfilled` 或 `rejected` 变化的两条路径，因此，我们实现两个函数来做这个事情。

```js
// 其他部分略…
class PromiseAplus {
  constructor() {
    // 其他部分略…
    const resolve = (value) => {
      this._PromiseState = FULFILLED
      this._PromiseResult = value
    }

    const reject = (reason) => {
      this._PromiseState = REJECTED
      this._PromiseResult = reason
    }
  }
  // 其他部分略…
}
```

那么这两个函数在什么时候被调用呢？回想我们平时使用 Promise 的方式，答案就是用户创建 Promise 的时候，在适当时机手动调用。

于是，我们需要将这两个函数暴露出去给 Promise 创建方调用，具体为，用户创建 promise 的时候，传入一个执行器，在执行器中，以参数形式注入这两个函数。

实现代码：

```js
// 其他部分略…
class PromiseAplus {
  // 传入执行器
  constructor(executor) {
    // 其他部分略…
    const resolve = (value) => {
      this._PromiseState = FULFILLED
      this._PromiseResult = value
    }

    const reject = (reason) => {
      this._PromiseState = REJECTED
      this._PromiseResult = reason
    }

    // resolve, reject 作为参数注入执行器，暴露给使用者自行调用
    // 此处 executor 本身也可能抛异常，如果捕获到异常，直接 reject 即可。
    try {
      executor(resolve, reject)
    }
    catch(error) {
      reject(error)
    }
  }
  // 其他部分略…
}
```

实现了 Promise 的构造方法，接着我们实现另一个核心，即 `then` 方法，需要实现以下要点：
1. 返回一个 promise
2. 通过回调可以访问 value 或 reason

```js
// 其他部分略…
class PromiseAplus {
  // 其他部分略…
  then(onFulfilled, onRejected) {
    // then 每次调用，都必须返回一个 promise
    return new PromiseAplus((resolve, reject) => {
      // 如果当前 promise 已经 fulfilled，则可以访问到 value
      if (this._PromiseState === FULFILLED) {
        if (typeof onFulfilled === 'function') {
          onFulfilled(this._PromiseResult)
        }
      }
      // 如果当前 promise 已经 rejected，则可以访问到 reason
      if (this._PromiseState === REJECTED) {
        if (typeof onRejected === 'function') {
          onRejected(this._PromiseResult)
        }
      }
      // 如果当前 promise 为 PENDING，则一会再考虑…
    })
  }
  // 其他部分略…
}
```

试着使用一下：

```js
const promise1 = new PromiseAplus((resolve, reject) => {
  resolve('fulfilled!')
})
const promise2 = new PromiseAplus((resolve, reject) => {
  reject('rejected!')
})
const onFulfilled = value => console.log('value is: ', value)
const onRejected = reason => console.log('reason is: ', reason)

promise1.then(onFulfilled, onRejected) // 打印：'value is:  fulfilled!'
promise2.then(onFulfilled, onRejected) // 打印：'reason is:  rejected!'
```

如此已经有了 Promise 的雏形了。

接着继续补完 `then` 在 `pending` 状态的时的处理逻辑。根据规范，两个回调不能在 promise fulfilled 或 rejected 之前被调用，那就意味着，我们要先将现场信息保存下来，留待 promise 状态变化的时候，再取出执行。

我们可以用一个闭包来封装保存现场信息和封装操作，然后新增一个数组属性来存放这些闭包，最后再实现一个遍历执行的函数。

代码如下：

```js
// 其他代码略…
class PromiseAplus {
  constructor(executor) {
    // 其他代码略…

    // 新增一个属性用来保存将要执行的 then 回调操作
    this._callbacks = []

    // 新增一个遍历执行所有保存的 then 回调的函数
    const walkCallbacks = () => {
      const len = this._callbacks.length
      for (let i = 0; i < len; i += 1) {
        this._callbacks[i]()
      }
      this._callbacks = []
    }

    // 改造 resolve 和 reject 函数，在状态从 pending 转换成 fulfilled 或 rejected 的时候，执行暂存的 then 回调操作
    const resolve = (value) => {
      this._PromiseState = FULFILLED
      this._PromiseResult = value
      walkCallbacks()
    }
    const reject = (reason) => {
      this._PromiseState = REJECTED
      this._PromiseResult = reason
      walkCallbacks()
    }

    // 其他代码略…
  }

  // 重新改造 then 方法的实现
  then(onFulfilled, onRejected) {
    return new PromiseAplus((resolve, reject) => {
      // 用一个闭包封装好当前状态和将要做的操作
      const resolution = () => {
        if (this._PromiseState === FULFILLED) {
          if (typeof onFulfilled === 'function') {
            onFulfilled(this._PromiseResult)
          }
        }
        if (this._PromiseState === REJECTED) {
          if (typeof onRejected === 'function') {
            onRejected(this._PromiseResult)
          }
        }
      }

      // 然后根据状态是否 PENDING，决定立即应用操作还是先保存起来延迟执行
      if (this._PromiseState === PENDING) {
        this._callbacks.push(resolution)
      }
      else {
        resolution()
      }
    })
  }
}
```


再测试下效果：

```js
const promise1 = new PromiseAplus((resolve, reject) => {
  setTimeout(() => {
    resolve('fulfilled!')
  }, 2000)
})
const promise2 = new PromiseAplus((resolve, reject) => {
  setTimeout(() => {
    reject('rejected!')
  }, 2000)
})
const onFulfilled = value => console.log('value is: ', value)
const onRejected = reason => console.log('reason is: ', reason)
promise1.then(onFulfilled, onRejected) // 2s 后打印：'value is:  fulfilled!'
promise2.then(onFulfilled, onRejected) // 2s 后打印：'reason is:  rejected!'
```

至此，我们实现了通过回调去访问 promise 的 value 和 reason 的特性。

根据规范，我们这两个回调的作用，除了访问 promise 的 value 和 reason，还应当产生新的结果，用来作为 `then` 返回的 promise 的结果参与决议流程。

所以继续改造，以便可以利用回调的返回值：

```js
// 其他代码略…
class PromiseAplus {
  // 其他代码略…
  then(onFulfilled, onRejected) {
    return new PromiseAplus((resolve, reject) => {
      // 改造 resolution 的实现，将两个回调的结果作为当前 Promise 的 value 或 reason
      const resolution = () => {
        const fn = this._PromiseState === FULFILLED ? onFulfilled : onRejected
        try {
          const result = fn(this._PromiseResult)
          resolve(result)
        }
        catch (error) {
          reject(error)
        }
      }
      // 其他代码略…
    })
  }
}
```

改造到这个阶段，就已经实现了 Promise 的最核心部分了。

然后我们再考虑下这种使用情况：

```js
new Promise((resolve, reject) => {
  resolve('resolve!')
})
.then() // 什么也不干
.then((value) => console.log(value)) // 这里能拿到 `'resolve!'` 这个 value。
```

这就是 Promise 的值的传递特性，我们接着实现这一点。

可以从 `onFulfilled` 和 `onRejected` 着手，在传入非函数的时候，我们自己构造一个函数作为默认实现，将 value 或 reason 原样传递下去即可。

```js
// 其他代码略…
class PromiseAplus {
  // 其他代码略…
  then(onFulfilled, onRejected) {
    return new PromiseAplus((resolve, reject) => {
      // 改造 resolution 的实现
      const resolution = () => {
        // 如果没有实现 onFulfilled | onRejected 逻辑，则使用透传实现
        const fn = this._PromiseState === FULFILLED
          // value 传递
          ? typeof onFulfilled === 'function' ? onFulfilled : (value => value)
          // reason 传递
          : typeof onRejected === 'function' ? onRejected : (reason => { throw reason })

        try {
          const result = fn(this._PromiseResult)
          resolve(result)
        }
        catch (error) {
          reject(error)
        }
      }
      // 其他代码略…
    })
  }
}
```

<!--
完整代码：
```js
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'
const PENDING = 'pending'
class PromiseAplus {
  constructor(executor) {
    this._PromiseState = PENDING
    this._PromiseResult = undefined
    this._callbacks = []

    const walkCallbacks = () => {
      const len = this._callbacks.length
      for (let i = 0; i < len; i += 1) {
        this._callbacks[i]()
      }
      this._callbacks = []
    }

    const resolve = (value) => {
      this._PromiseState = FULFILLED
      this._PromiseResult = value
      walkCallbacks()
    }
    const reject = (reason) => {
      this._PromiseState = REJECTED
      this._PromiseResult = reason
      walkCallbacks()
    }

    try {
      executor(resolve, reject)
    }
    catch(error) {
      reject(error)
    }
  }
  then(onFulfilled, onRejected) {
    return new PromiseAplus((resolve, reject) => {
      const resolution = () => {
        const fn = this._PromiseState === FULFILLED
          ? typeof onFulfilled === 'function' ? onFulfilled : (value => value)
          : typeof onRejected === 'function' ? onRejected : (reason => { throw reason })

        try {
          const result = fn(this._PromiseResult)
          resolve(result)
        }
        catch (error) {
          reject(error)
        }
      }
      if (this._PromiseState === PENDING) {
        this._callbacks.push(resolution)
      }
      else {
        resolution()
      }
    })
  }
}
```
-->

这样就可以确保 promise 的 value 或 reason 能正确传递下去。

之后再来实现最复杂的部分，promise 的决议过程（Resolution Procedure）。

需要注意的要点比较多，因此下面直接在代码中，使用注释来详细说明。

```js
class PromiseAplus {
  constructor(executor) {
    // 其他代码略…

    const resolve = (value) => {
      // 状态变为 fulfilled 或 rejected 后，就不能再变化了
      if (this._PromiseState !== PENDING) return

      // 如果 `promise` 和 `value` 是同一个对象引用，那么直接抛 `TypeError` 错误（避免循环）。 
      if (value === this) {
        throw new TypeError('Chaining cycle detected for promise #<Promise>')
      }

      // 如果是对象（不含 null）或者函数，有可能是 thenable，
      // 为了跟其他 Promise/A+ 的实现互操作，需要特殊处理
      if (typeof value === 'function' || (typeof value === 'object' && value !== null)) {
        // 该 flag 用来避免多次 resolve 或 reject，
        // 在尝试多次调用的时候，只处理第一次，后续的都忽略
        let ignore = false

        try {
          // 访问 then 出错也应当 reject
          const then = value.then

          // thenable
          if (typeof then === 'function') {
            // 包装当前 promise 的 resolve 函数，加入 ignore 控制
            const resolvePromise = y => {
              if (ignore) return
              ignore = true
              resolve(y)
            }
            // 包装当前 promise 的 reject 函数，加入 ignore 控制
            const rejectPromise = r => {
              if (ignore) return
              ignore = true
              reject(r)
            }

            // 对于 thenable（也包括本自定义 promise 类或其他 promise 类）
            // 需要等待其被 resolve 或 reject，才能 resolve 或 reject 当前 promise
            return then.call(value, resolvePromise, rejectPromise)
          }
        }
        catch(error) {
          // 如果出现错误，则 reject promise，需要避免多次 reject
          if (!ignore) reject(error)
          return
        }
      }

      // 对于非 thenable 的对象或函数，以及其他类型的值，则直接 resolve
      this._PromiseState = FULFILLED
      this._PromiseResult = value

      // resolve 之后，可以开始调用所有 then 对应的 promise 的 resolution 逻辑了
      walkCallbacks()
    }

    const reject = (reason) => {
      // 状态变为 fulfilled 或 rejected 后，就不能再变化了
      if (this._PromiseState !== PENDING) return

      this._PromiseState = REJECTED
      this._PromiseResult = reason

      // reject 之后，可以开始调用所有 then 对应的 promise 的 resolution 逻辑了
      walkCallbacks()
    }

    // 其他代码略…
  }
}

```

这样一来，我们的 Promise 就具备了处理 promise 嵌套的能力了。


再看看规范，我们还有一点没有做到，那就是 `then` 的两个回调，都需要异步执行。

在浏览器的 Promise 实现中，这是利用微任务来做的。我们参考浏览器的实现，接下来就尝试将调用异步化。

首先实现一个可以异步调用函数的辅助函数：

```js
// 用 IIFE 封装，里面根据情况返回对应的实现。
const queueMicrotask = (function() {
  // 如果当前环境原生支持 global.queueMicrotask，则直接使用
  if (this.queueMicrotask) return this.queueMicrotask

  // 对于浏览器环境，使用 MutationObserver 来模拟
  if (this.document && this.MutationObserver) {
    let seed = 0
    const $el = document.createElement('div')
    const queue = () => $el.setAttribute('change', ++seed)

    return fn => {
      const observer = new MutationObserver(() => {
        fn()
        observer.disconnect()
      })
      observer.observe($el, { attributes: true })
      queue()
    }
  }

  // 对于 node 环境，使用 process.nextTick
  if (this.process) {
    return fn => process.nextTick(fn)
  }

  // fallback 为 setTimeout
  return fn => setTimeout(fn, 0)
})()
```

然后改造我们的 `then` 方法，添加上异步调用的代码：

```js
class PromiseAplus {
  // 其他代码略…
  then(onFulfilled, onRejected) {
    return new this.constructor((resolve, reject) => {
      const resolution = () => {
        const fn = this._PromiseState === FULFILLED
          ? typeof onFulfilled === 'function' ? onFulfilled : defaultOnFulfilled
          : typeof onRejected === 'function' ? onRejected : defaultOnRejected

        // 异步化，在微任务中调用 then 的 onFulfilled，根据结果 resolve OR reject
        queueMicrotask(() => {
          try {
            const result = fn(this._PromiseResult)
            resolve(result)
          }
          catch(error) {
            reject(error)
          }
        })
      }

      if (this._PromiseState === PENDING) {
        this._callbacks.push(resolution)
      }
      else {
        resolution()
      }
    })
  }
  // 其他代码略…
}
```

至此，我们就算彻底完成一个 Promise/A+ 的实现了。

对上面所写的代码，略作整理调整，完整的代码如下：

```js
const queueMicrotask = (function() {
  if (this.queueMicrotask) return this.queueMicrotask
  if (this.document && this.MutationObserver) {
    let seed = 0
    const $el = document.createElement('div')
    const queue = () => $el.setAttribute('change', ++seed)

    return fn => {
      const observer = new MutationObserver(() => {
        fn()
        observer.disconnect()
      })
      observer.observe($el, { attributes: true })
      queue()
    }
  }
  if (this.process) {
    return fn => process.nextTick(fn)
  }
  return fn => setTimeout(fn, 0)
})()

const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'
const PENDING = 'pending'

const defaultOnFulfilled = v => v
const defaultOnRejected = e => { throw e }

class PromiseAplus {
  constructor(executor) {
    this._PromiseState = PENDING
    this._PromiseResult = undefined
    this._callbacks = []

    const walkCallbacks = () => {
      const len = this._callbacks.length
      for (let i = 0; i < len; i += 1) {
        this._callbacks[i]()
      }
      this._callbacks = []
    }

    const resolve = (value) => {
      if (this._PromiseState !== PENDING) return
      if (value === this) {
        throw new TypeError('Chaining cycle detected for promise #<Promise>')
      }

      if (typeof value === 'function' || (typeof value === 'object' && value !== null)) {
        let ignore = false
        try {
          const then = value.then
          if (typeof then === 'function') {
            const resolvePromise = y => {
              if (ignore) return
              ignore = true
              resolve(y)
            }
            const rejectPromise = r => {
              if (ignore) return
              ignore = true
              reject(r)
            }
            return then.call(value, resolvePromise, rejectPromise)
          }
        }
        catch(error) {
          if (!ignore) reject(error)
          return
        }
      }

      this._PromiseState = FULFILLED
      this._PromiseResult = value
      walkCallbacks()
    }
  
    const reject = (reason) => {
      if (this._PromiseState !== PENDING) return

      this._PromiseState = REJECTED
      this._PromiseResult = reason
      walkCallbacks()
    }
    try {
      executor(resolve, reject)
    }
    catch(error) {
      reject(error)
    }
  }

  then(onFulfilled, onRejected) {
    return new this.constructor((resolve, reject) => {
      const resolution = () => {
        const fn = this._PromiseState === FULFILLED
          ? typeof onFulfilled === 'function' ? onFulfilled : defaultOnFulfilled
          : typeof onRejected === 'function' ? onRejected : defaultOnRejected
        queueMicrotask(() => {
          try {
            const result = fn(this._PromiseResult)
            resolve(result)
          }
          catch(error) {
            reject(error)
          }
        })
      }
      if (this._PromiseState === PENDING) {
        this._callbacks.push(resolution)
      }
      else {
        resolution()
      }
    })
  }
}
```

感兴趣的可以尝试去浏览器跑一跑代码验证一下。

不过要证明该实现确实符合 Promise/A+ 的所有要求，还得跑一遍官方的测试集才算。

下面说明下怎么做测试。

---

## 测试

首先是初始化一个 npm 包，然后安装测试依赖：

```bash
npm install promises-aplus-tests --save-dev
```

然后书写测试入口文件，在这个入口文件中，调用官方测试函数，并传入我们实现好的 Promise 即可。

官方测试函数为了兼容各种不同的实现，使用了适配器模式，需要我们自己按照要求写好适配器，再传入测试。

适配器需要实现三个接口：

1. resolved 方法，用于返回一个指定 value 的处于 fulfilled 状态的 promise。
2. rejected 方法，用于返回一个指定 reason 的处于 rejected 状态的 promise。
3. deferred 方法，返回一个对象，这个对象中包含三个字段，分别为：  
  - promise，为一个 promise 对象。
  - resolve，为一个函数，调用可以用指定的 value 来 resolve 掉上述 promise。
  - reject，为一个函数，调用可以用指定的 reason 来 reject 掉上述 promise。

下面示范该入口文件的写法：

```js
// 引入测试方法
const promisesAplusTests = require("promises-aplus-tests")

// 引入自己实现的 Promise
const Promise = require('./promise')

// 写一个测试需要的适配器
// 必须实现三个方法：rejected，resolved，deferred
const adapter = {
  // 该方法要求传入一个 resolve 的值，返回一个对应的 fulfilled 状态的 Promise
  resolved: function(value) {
    return Promise.resolve(value)
  },

  // 该方法要求传入一个 reject 的原因，返回对应一个 rejected 状态的 Promise
  rejected: function(reason) {
    return Promise.reject(reason)
  },
  
  // 该方法要求返回一个对象，
  // 对象包含三个属性：
  // 1. promise（一个 Promise 对象）
  // 2. resolve（resolve 上述 promise 的一个方法）
  // 3. reject（reject 上述 promise 的一个方法）
  deferred: function() {
    let resolve
    let reject
    const promise = new Promise((_resolve, _reject) => {
      resolve = _resolve
      reject = _reject
    })
  
    return {
      promise,
      resolve,
      reject,
    }
  }
}

// 然后调用该函数即可开始测试
promisesAplusTests(adapter, function (err) { /* All done; output is in the console. Or check `err` for number of failures. */ })

```

写好入口文件后，我们还需要改造下我们实现的 Promise，将其导出成一个 cmd 包。

只需要简单地在末尾加上导出语句即可：

```js
// 其余代码略…
module.exports = PromiseAplus
```
最后就是执行测试了，简单的使用 node 执行该测试入口文件即可，执行过程屏幕将会打印测试通过跟失败的记录。如果实现正确，将会通过全部 872 项测试。

---

## 改进完善

为了更接近我们实际使用的 Promise，还有一些实例方法和静态方法需要实现，我们在上面实现好的 PromiseAplus 基础上，通过继承来扩展我们的新功能。

```js
class MyPromise extends PromiseAplus {
}
```

### catch 实例方法

`catch` 方法只是 `then` 方法的一个特例，实现比较简单：

```js
class MyPromise extends PromiseAplus {
  // 其他代码略…
  catch(onRejection) {
    return this.then(null, onRejection)
  }
}
```
  
### finally 实例方法

`finally` 类似 try catch 的 `finally`，不管 promise 的状态切换成哪种，均为被调用。

```js
class MyPromise extends PromiseAplus {
  // 其他代码略…
  finally(onFinally) {
    return this.then(
      value => this.constructor.resolve(onFinally()) .then(() => value),
      reason => this.constructor.resolve(onFinally()).then(() => { throw reason })
    )
  }
}
```

### resolve 静态方法

只需要简单地封装下现有的方法：

```js
class MyPromise extends PromiseAplus {
  static resolve(value) {
    if (value instanceof this.prototype.constructor) {
      return value
    }
    return new this.prototype.constructor(resolve => resolve(value))
  }
  // 其他代码略…
}
```

### reject 静态方法

```js
class MyPromise extends PromiseAplus {
  // 其他代码略…
  static reject(error) {
    return new this.prototype.constructor((_, reject) => reject(error))
  }
  // 其他代码略…
}
```

### all 静态方法

```js
class MyPromise extends PromiseAplus {
  // 其他代码略…
  static all(iterable) {
    const promises = [...iterable]
    return new MyPromise((resolve, reject) => {
      const results = []
      const length = promises.length
      if (!length) return resolve(results)

      let stop = false
      let count = length
      for (let i = 0; i < length; i += 1) {
        MyPromise.resolve(promises[i]).then(
          value => {
            results[i] = value
            if (!--count) {
              resolve(results)
            }
          },
          reason => {
            if (stop) return
            stop = true
            reject(reason)
          }
        )
      }
    })
  }
  // 其他代码略…
}
```

### race 静态方法

```js
class MyPromise extends PromiseAplus {
  // 其他代码略…
  static race(iterable) {
    const promises = [...iterable]
    return new MyPromise((resolve, reject) => {
      const length = promises.length
      // 返回永远 pending 的 promise
      if (!length) return new MyPromise(() => {})

      let stop = false
      for (let i = 0; i < length; i += 1) {
        MyPromise.resolve(promises[i]).then(
          value => {
            if (stop) return
            stop = true
            resolve(value)
          },
          reason => {
            if (stop) return
            stop = true
            reject(reason)
          }
        )
      }
    })
  }
  // 其他代码略…
}
```

### allSettled 静态方法

```js
class MyPromise extends PromiseAplus {
  // 其他代码略…
  static allSettled(iterable) {
    const promises = [...iterable]
    return new MyPromise((resolve, reject) => {
      const results = []
      const length = promises.length
      if (!length) return resolve(results)

      let count = length
      for (let i = 0; i < length; i += 1) {
        MyPromise.resolve(promises[i]).then(
          value => {
            results[i] = {
              status: 'fulfilled',
              value
            }
            if (!--count) {
              resolve(results)
            }
          },
          reason => {
            results[i] = {
              status: 'rejected',
              value: reason,
            }
            if (!--count) {
              resolve(results)
            }
          }
        )
      }
    })
  }
  // 其他代码略…
}
```

---

全文完
