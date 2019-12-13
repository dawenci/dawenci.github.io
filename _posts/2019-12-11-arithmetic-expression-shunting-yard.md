---
layout: post
title:  "算术表达式解析系列之逆波兰表示法"
categories: JavaScript
tags: [JavaScript]
---

假如我们有这样的一个需求，接受一个用户录入的算术表达式，解析并计算出结果。

其中，这个表达式中，可以存在：

1. 数，表示参与计算的对象
2. 运算符（可以是自定义的运算符），表示不同的计算
3. 括号，表示对计算顺序的干预

那可以怎么做？

本系列文章，将提供几种不同的解决方案。本篇将介绍一种常见的方案：“逆波兰表示法”。

首先我们先了解一些概念。

## 中缀表示法

中缀表示法，应该是我们最常接触的算术表示法了。该表示法，操作符位于两个操作数中间，因此而得名。

【例 1】：

```
(10 + 15 - 20) * 30 / 40 ^ 2
```

这就是一个使用中缀表示法的例子。

在中缀表示法中，括号可以改变一个表达式的计算顺序。

`(1 - 1) - 1` 和 `1 - (1 - 1)` 是完全不同的。所以， `1 - 1 - 1` 这样的表达式，对于计算机来说，是存在
解析的歧义的，处理起来相对麻烦。


## 逆波兰表示法

逆波兰表示法中，操作符放在操作数的后面，因此也叫做后缀表示法，上面的 【例 1】，可以这样等价地表示：

```
10 15 + 20 - 30 * 40 2 ^ /
```

虽然跟我们的数学经验有着很大的差别，但是在计算机处理方面，却有着其独特的优势。逆波兰表示法的一个特点是，只要元数固定，无需括号，也能无歧义地解析。

就像【例 2】中的 `(1 - 1) - 1` ，可以用 `1 1 - 1 -` 来表示，而 `1 - (1 - 1)` 则为 `1 1 1 - -`。

逆波兰表示法中，运算顺序才是决定运算顺序的因素，而非括号。

而且，逆波兰表示法的计算规则比较简单，利用栈就可以高效地运算，所以，简直就是为计算机量身定制。

为了更好地说明逆波兰表示法，下面开始实现一个简单的算术表达式计算器，通过代码来加深理解。

首先设定一些小目标：

1. 数值支持，为了方便，只支持正数
2. 运算符支持，`+`（加）, `-`（减）, `*`（乘）, `/`（除）, `%`（模）, `^` （指数）等运算
3. 圆括号支持

同时还是为了简单起见，不实现表达式 `tokenize` 方法，直接人工输入分好词的 token（下文使用记号一词代替 token）数组。

例如上述【例 1】，直接使用 `["10", "15", "+", "20", "-", "30", "*", "40", "2", "^", "/"]` 这种形式的
输入来表示。

然后是程序的大致执行流程：

1. 将中缀表示法的记号序列，转换成逆波兰表示法的记号序列，即 `["10", "15", "20", "+", "30", "40",
   "2", "^", "*", "/", "-"]` 这种形式
2. 实现转换后的逆波兰记号序列的计算，如录入上述记号流的运算结果应为 `0.09375`。


## 开始实现

首先定义支持的运算符，此处需要关注运算符的优先级和结合性。

优先级决定了不同运算符之间的运算顺序优先级，例如，加法和乘法，加法的优先级低于乘法，所以 `10 + 10 * 10`，会解析成 `10 + (10 * 10)` 而非 `(10 + 10) * 10`。

而结合性决定了相同的运算符连续出现时操作数的组合规则，例如减号是左结合的，所以，`10 - 10 - 10` 要先
运算左边的操作数，再将结果和后面的操作数结合，相当于 `(10 - 10) - 10`。
而右结合则相反，比如我们这里规定 `^` 运算为右结合，那么 `10 ^ 10 ^ 10`，就相当于 `10 ^ (10 ^ 10)`。

下面代码用一个对象来配置运算符，key 为运算符，value 为运算符的优先级和结合性。

```js
const operators = {
  '+': {
    precedence: 1,
    associativity: 'left',
  },
  '-': {
    precedence: 1,
    associativity: 'left',
  },  
  '*': {
    precedence: 2,
    associativity: 'left',
  },
  '/': {
    precedence: 2,
    associativity: 'left',
  },
  '%': {
    precedence: 2,
    associativity: 'left',
  },
  '^': {
    precedence: 3,
    associativity: 'right',
  },
}
```

定义好运算符之后，再定义一对分组用的符号，用于中缀表示法中调整优先级，此处我们按照习惯使用圆括号。

```js
const groupStart = '('
const groupEnd = ')'
```

然后，开始实现中缀表示法转逆波兰表示法，此处我们使用一个经典的算法，“调度场算法（Shunting Yard Algorithm）” 来进行。

### 调度场算法

该算法需要两个数组，一个放输出（即结果），一个当作栈使用，用来存放算法过程的一些运算符。

算法执行过程要点如下：

从左到右扫描中缀表示法的每个元素，根据遇到的不同元素，执行对应的逻辑：

- case 1: 当前为操作数  
  直接输出操作数

- case 2: 当前为运算符，且运算符栈空  
  运算符直接入栈

- case 3: 当前为左括号，或运算符栈顶是左括号  
  运算符直接入栈  

- case 4: 当前为右括号  
  运算符栈一直弹出并输出，直至遇到左括号。左括号弹出不输出，右括号舍弃不输出  

- case 5: 当前为运算符，且运算符栈非空
  若栈顶是左括号，则停止比较，当前运算符直接入栈
  若当前运算符优先级大于栈顶运算符，则停止比较，当前运算符直接入栈
  若当前运算符与栈顶运算符相同，并且当前运算符是右结合的，则停止比较，当前运算符直接入栈
  其他情况，将栈顶运算符出栈输出，回到 case 5，直至栈空时，将当前运算符入栈

扫描完毕之后，将运算符栈中的所有剩余运算符输出

下面是对应的代码实现

```js

// 帮助函数：获取栈顶元素
const stackTop = (stack) => stack[stack.length - 1]

// 帮助函数：是否操作数
const isNumber = (n) => /\d/.test(n)

// 转换函数
// infixTokens 即为 ['1', '+', '1'] 形式的中缀表示法的记号流
function toRpn(infixTokens) {
  const len = infixTokens.length
  // 存放输出
  const output = []
  // 运算符栈
  const opStack = []

  let index = -1
  while (++index < len) {
    const token = infixTokens[index]

    // case 1
    if (isNumber(token)) {
      output.push(token)
      continue
    }

    // case 2
    // case 3
    if (!opStack.length || groupStart === token) {
      opStack.push(token)
      continue
    }

    // case 4
    if (groupEnd === token) {
      while (opStack.length) {
        const top = opStack.pop()
        if (groupStart === top) break
        output.push(top)
      }
      continue
    }

    // assert operator
    const op = operators[token]
    if (!op) throw new Error(`运算符(${token})未定义`)

    // case 5
    while (opStack.length) {
      if (groupStart === stackTop(opStack)) break
      const top = stackTop(opStack)
      const topOp = operators[top]
      if (op.precedence - topOp.precedence > 0) break
      if (token === top && topOp.associativity === 'right') break
      output.push(opStack.pop())
    }

    opStack.push(token)
  }

  // 将栈中所有剩余运算符输出
  while (opStack.length) {
    output.push(opStack.pop())
  }

  return output
}
```

通过上面的 toRpn 函数，即可将中缀表示法转换成逆波兰表示法。

```js
const rpnTokens = toRpn(["10", "15", "+", "20", "-", "30", "*", "40", "2", "^", "/"])
console.log(rpnTokens) // ["10", "15", "20", "+", "30", "40", "2", "^", "*", "/", "-"]
```

有了后缀表示法记号流之后，我们接下来开始实现计算逻辑。

### 实现计算

逆波兰表示法的计算，需要利用栈。
过程中比较简单，扫描表达式，遇到操作数，则入栈，遇到运算符，则从栈中取出操作数，执行运算。


开始之前，先改进下之前的 operators 定义，为每个运算符指定对应的计算语义，实现运算逻辑。


```js
const operators = {
  '+': {
    precedence: 1,
    associativity: 'left',
    calc: (a, b) => a + b
  },
  '-': {
    precedence: 1,
    associativity: 'left',
    calc: (a, b) => a - b
  },  
  '*': {
    precedence: 2,
    associativity: 'left',
    calc: (a, b) => a * b
  },
  '/': {
    precedence: 2,
    associativity: 'left',
    calc: (a, b) => a / b
  },
  '%': {
    precedence: 2,
    associativity: 'left',
    calc: (a, b) => a % b
  },
  '^': {
    precedence: 3,
    associativity: 'right',
    calc: (a, b) => Math.pow(a, b)
  },
}
```

注意上面的代码中，每个运算符的定义中都多了一个 `calc` 函数，该函数接受两个操作数，返回对应的计算结果。

然后开始运算的逻辑实现。


```js
// 参数 rpnTokens 为转换好的逆波兰表示法记号流
function calc(rpnTokens) {
  const stack = []
  const len = rpnTokens.length
  let position = -1
  let result = 0

  for (let i = 0; i < len; i++) {
    const token = rpnTokens[i]

    if (isNumber(token)) {
      position++
      stack[position] = token
      continue
    }

    const op = operators[token]
    if (!op) throw new Error(`运算符(${token})未定义`)    

    // 操作数字符串转数值
    const left = +stack[position - 1]
    const right = +stack[position]
    stack[--position] = op.calc(left, right)
  }

  result = stack[position]
  return result
}
```

如此便完成了计算逻辑的实现。

```
calc(rpnTokens) // 0.09375
```

可以看到代码如期望地计算出了结果。


### 改进

理解了上面的算法逻辑后，就可以根据需要， 增加支持的运算符、修改分组符号、甚至增加其他语法成分的支持了。

如果感兴趣，可以在我的 [Github](https://github.com/dawenci/rpn) 上获得对应本文一份完整的参考代码。


## 下篇预告

算术表达式解析系列之优先级爬升法
