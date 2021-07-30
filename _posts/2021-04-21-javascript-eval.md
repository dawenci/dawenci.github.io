---
layout: post
title:  "JavaScript 的 eval"
categories: JavaScript
tags: [JavaScript]
---


## eval 的使用方式

`eval` 可以接受一个字符串，并解析执行，例如：

```js
var a = 1;
eval('a + 1') // 2

eval('{ foo: 123 }') // 123，代码块
eval('({ foo: 123 })') // { foo: 123 }，对象字面量
```


### 严格模式

通常 `eval` 应当只在严格模式下使用。 
非严格模式下，使用 `eval` 声明的变量，会成为当前作用域中的一个本地的变量，可以被周边的代码访问。而严格模式中，默认使用的是全局作用域。

```js
var foo = 'global'

;(function f() {
  eval(`var foo = 'local'`);
  console.log(foo); // 'local'
})()

;(function f() {
  'use strict'
  eval(`var foo = 'local'`);
  console.log(foo); // 'global'
})()

```

因为 JavaScript 使用的是词法作用域，为了实现动态作用域的效果（方便读取到栈帧中的变量），规范中进行了一些特殊的规定，将 `eval`  的调用区分为直接调用和间接调用。其中直接调用的话，就显式将当前栈帧传入，以达到动态作用域的效果。



## 直接调用及间接调用

`eval` 有两种调用方式，分别为直接调用、间接调用。

直接调用，是这样的一种调用，调用表达式中，成员表达式计算结果为一个引用，该引用的名称为 `eval` ，并且对该引用执行抽象操作 `GetValue` 的结果为全局的 `eval` 函数，此即为直接调用。

在规范中描述如下（https://262.ecma-international.org/5.1/#sec-15.1.2.1.1）：

> A direct call to the eval function is one that is expressed as a *CallExpression* that meets the following two conditions:
>
> The [Reference](https://262.ecma-international.org/5.1/#sec-8.7) that is the result of evaluating the *MemberExpression* in the *CallExpression* has an environment record as its base value and its reference name is "**eval**".
>
> The result of calling the abstract operation [GetValue](https://262.ecma-international.org/5.1/#sec-8.7.1) with that [Reference](https://262.ecma-international.org/5.1/#sec-8.7) as the argument is the standard built-in function defined in [15.1.2.1](https://262.ecma-international.org/5.1/#sec-15.1.2.1).



下面这种形式的调用都是直接调用：

```js
// 以下 eval，均为 “引用”，名称为 "eval"
eval('foo')
;(eval)('foo')
;(((eval)))('foo')
;(function() { return eval('foo') })()
eval('eval("foo")')
;(function(eval) { return eval('foo'); })(eval)
with({ eval: eval }) eval('foo')
with(window) eval('foo')
```

而下面这种形式的调用，都是间接调用：

```js
// 以下 eval，要么是一个值而非一个引用，要么引用的名称不是 'eval'。
window.eval('foo')
eval.call(null, 'foo')
;(0, eval)('foo')
var _eval = eval
_eval('foo')
```



直接调用、间接调用链接到的作用域不同，其中，直接调用，会使用本地作用域，而间接调用，则使用全局作用域：

```js
var foo = 'global'
;(function f() {
  'use strict'
	var foo = 'local'
  console.log(eval('foo')) // 'local'
	console.log(window.eval('foo')) // 'global'
	console.log(eval.call(null, 'foo')) // 'global'
	console.log(eval.apply(null, ['foo'])) // 'global'
	console.log(eval.bind(null, 'foo')()) // 'global'
	console.log(eval.bind(null)('foo')) // 'global'
	var _eval = eval
	console.log(_eval('foo')) // global
	console.log(('any', eval)('foo')) // global
})()

```

间接调用，结果总是非严格的，无论是否使用了 `'use strict'` 。

```js
(function strictMode() {
  'use strict';
  var code = '(function () { return this }())'
  console.log(eval(code)); // undefined
  console.log(eval.call(null, code)); // window
})()
```



最后，`eval` 使用间接调用，性能是远高于直接调用的。



## new Function()

类似与 `eval()`，使用 `new Function()` 创建的函数，作用域是全局作用域。例如：

```js
var foo = 'global'
;(function strictMode() {
  'use strict'
  var foo = 'local'
  var f = new Function('return foo');
  console.log(f()) // global
})()
```

以及，这些函数也默认为非严格模式的：

```js
(function strictMode() {
  'use strict'
  var sloppy = new Function('return this')
  console.log(sloppy()) // window
  var strict = new Function('"use strict"; return this');
  console.log(strict()) // undefined
})()
```



## 参考

[Evaluating JavaScript code via eval() and new Function()](https://2ality.com/2014/01/eval.html)  
[Global eval. What are the options?](http://perfectionkills.com/global-eval-what-are-the-options/)  
[ECMAScript® Language Specification](https://262.ecma-international.org/5.1/#sec-15.1.2.1)  
