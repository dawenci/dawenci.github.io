---
layout: post
title:  "JavaScript 变量赋值的问题"
categories: JavaScript
tags: [JavaScript]
---

最近看到有人在讨论一个关于 JavaScript 赋值的问题，代码如下：

```js
var a = {n: 1}
var b = a
a.x = a = {n: 2}

console.log(a.x) // undefined
console.log(b.x) // {n: 2}
```

代码看着有点绕，这个结果很多人表示不能理解。
为了说明这个问题，下面便逐步详细分析。

为了化简式子，下面行文的一些约定：

1. 声明环境记录用 `ENV` 表示
2. `{n: 1}` 用 `N1` 表示
3. `{n: 2}` 用 `N2` 表示


分析开始：

执行完第一行，声明环境记录 `ENV` 里面存在绑定：
`ENV.a` --指向--> `{n: 1}`（后续用 `N1` 指代）

执行完第二行，声明环境记录 `ENV` 里面存在绑定：
`ENV.a` --指向--> `N1`
`ENV.b` --指向--> `N1`

执行第三行，将三部分分别来看：
1. 第一部分 `a.x` 是 `ENV.a.x`，即 `N1.x`
2. 第二部分 `a` 是 `ENV.a`，即 `N1`
3. 第三部分为 `{n: 2}`（后续用 `N2` 指代）

赋值右结合，相当于
`N1.x = (ENV.a = N2)`

先看第一个赋值动作的执行：
`ENV.a = N2`

执行后：
1. `ENV.a` --指向--> `N2`
2. `ENV.b` --指向--> `N1`
3. 表达式返回值为 `N2`

继续第二个赋值动作，将表达式的返回值 `N2` 赋值给第一部分：
`N1.x = N2`

此时：
1. `ENV.a` --指向--> `N2`，`N1` 现在为 `{n: 2, x: N2}`
2. `ENV.b` --指向--> `N1`
3. 表达式返回值为 `N2`

执行 `console.log(a.x)`:
此时 `ENV` 中的 `a` 为 `N2`，相当于：
`console.log(N2.x)`
而 `N2` 中没有 `x` 属性，所以结果：
`undefined`

执行 `console.log(b.x)`:
此时 `ENV` 中的 `b` 为 `N1`，相当于：
`console.log(N1.x)`
所以结果为 `N2`，即 `{n: 2}`


