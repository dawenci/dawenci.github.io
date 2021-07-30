---
layout: post
title:  "λ 演算（λ-calculus）笔记"
categories: FP
tags: [FP]
---

文中涉及到代码的地方，除了 λ 表达式，都尽量提供 Scheme 以及 JavaScript 代码做对照，以便加深理解。

先从基本概念开始。λ 演算包含构建 λ 项和对 λ 项执行操作。


## λ 项 (lambda terms) 构建语法

* 变量 ( variable )

  **expression -> variable**

* 应用 ( application )

  **expression -> expression expression**

* 抽象化 ( abstraction )

  **expression -> λ variable . expression**

> 为了不引起歧义，可以适当使用括号进行分组 ( grouping )：
>
> **expression -> (expression)** 


### 变量

可以理解为编程语言中的变量、标识符（一些用来绑定值的名称），例如 `a` , `b`, `c` 等等。


### 应用

应用是指将函数应用 ( applying ) 在实参 ( argument ) 上面。注连续多个应用是左结合的。

举例说明：

<!-- more -->

```
Lambda 表达式：
abs x
add x y
(add x) y
f (x y)
```

```scheme
;; Scheme:
(abs x)
((add x) y)
((add x) y)
(f (x y))
```

```javascript
// JavaScript:
abs(x)
add(x)(y)
(add(x))(y)
f(x(y))
```



### 抽象

就是函数抽象，基本语法：

```
Lambda 表达式：
λa.a
```

其中 `λ`是函数符号，代表函数开始。中间的圆点`.`是函数参数 ( parameters ) 与函数体 ( body ) 的分隔符号。圆点之前的 `a`代表函数的参数，圆点之后的 `a` 为函数体，函数体为表达式的返回结果。

这个例子中的函数含义是，输入一个 `a` ，原样返回 `a`。

```scheme
;; Scheme:
(lambda (a) a)
```

```javascript
// JavaScript:
a => a
```



再来看一些例子：

```
Lambda 表达式：
λa.b
λa.b x
λa.(b x)
(λa.b) x
λa.λb.a
λa.(λb.a)
```

```scheme
;; Scheme:
(lambda (a) b)
(lambda (a) (b x))
(lambda (a) (b x))
((lambda (a) b) x)
(lambda (a) (lambda (b) a))
(lambda (a) (lambda (b) a))
```

```javascript
// JavaScript:
a => b
a => b(x)
a => b(x)
(a => b)(x)
a => b => a
a => b => a
```



### 柯里化

上面介绍抽象化的时候，列举的函数都是单一参数的，如果我们需要多个参数的时候，可以通过柯里化来实现。

我们先用支持多参数的 JavaScript 以加法函数为例子说明：

```javascript
// add 为接受两个参数的函数
let add = (x, y) => x + y 
// 直接传入两个参数使用
add(2, 3) // => 5

// 柯里化：
// 传入一个参数，然后返回一个新的函数
// 再传一个参数，返回结果：
add = x => (y => x + y) // => function
let addTwo = add(2) // => function
addTwo(3) // => 5

// 直接调用：
add(2)(3) // => 5
```

用 Scheme 表示：

```scheme
;; 接受两个参数：
((lambda (x y)
   (+ x y)) 2 3) ;; => 5

;; 柯里化：
(((lambda (x)
    (lambda (y)
      (+ x y))) 2) 
 3)
```

因此，多参数的函数，用 λ-演算可以表达为：

```
λx.λy.x+y
```



## 化简操作（reduction operations）



### α-equivalence （α 等价）

α-equivalence 简单讲就是，参数名称可以随意重命名而不影响意义，只要参数、函数体一起对应修改即可。实际应用有些额外的限制，这里不展开。



应用该操作可以用于避免名称冲突。

例如：

```
Lambda 表达式：
λa.a
```

可以转换为：

```
Lambda 表达式：
λb.b
```

转换前后是等价的。





### β-reduction （β 化简）

β-reduction 就是应用函数，使用实参替换掉函数体中所有的形参，最后返回替换后的函数体部分这个过程。

例如：

```
Lambda 表达式：

( (λa.a) λb.λc.b ) (x) λe.f
    // 将 λb.λc.b 代入 a:
    (λb.λc.b) (x) λe.f
      // 将 x 代入 b:
      (λc.x) λe.f
        // 将 λe.f 代入 c:
        x
```

使用 Scheme 语法描述：

```scheme
;; Scheme:

((((lambda (a) a) (lambda (b) (lambda (c) b))) x) (lambda (e) f))
	;; 将 (lambda (b) (lambda (c) b)) 带入 a
    (((lambda (b) (lambda (c) b)) x) (lambda (e) f))
      ;; 将 x 代入 b
      ((lambda (c) x) (lambda (e) f))
        ;; 将 (lambda (e) f) 代入 c
        	x
```

使用 JavaScript 语法牧师：

```javascript
// JavaScript:

((a => a)(b => (c => b)))(x)(e => f)
  // 将 (b => (c => b)) 代入 a
  (b => (c => b))(x)(e => f)
	// 将 x 代入 b
	(c => x)(e => f)
	  // 将 (e => f) 代入 c
	  x
```





### η-conversion （η 转换）



两个函数如果在任意一致的参数中都有一致的结果时，则这两个函数可以认为是同一个函数。

当且仅当它们是同一个函数时，η 变换可以令 `λx.fx` 和 `f` 相互交换，只要 `x` 不是 `f` 中的自由出现。

> 自由出现（free occurrence）跟约束出现（bound occurrence）对应，是指作为自由变量（free variable）出现，与之相对的约束出现则为作为约束变量（bound variable）出现。



## 组合子（ combinator ）

组合子即为不包含自由变量 ( free variables ) 的函数。

如：

```
Lambda 表达式：

λa.a
λab.a
λabc.c(λe.b)
```

而包含自由变量的函数不是组合子，例如：

```
Lambda 表达式：

λa.b
λa.ab
```



## Y-组合子 ( Y-combinator )



#### 函数的不动点（ fixed point ）

首先，我们得先了解一个重要的前置概念，函数的不动点。

函数 `f` 的不动点 (fixed point)，是这样的一些值，将 `f` 应用在这些值上，返回的结果等于这些值本身。比如： `f(n) = n`， `n` 就是函数 `f` 的一个不动点。



Y-组合子是一个用于计算高阶函数的不动点的函数。即 `Y(f)` 的结果是 `f` 的不动点。即，假设函数 `f` 的不动点为 `n`，那么有：`f(Y(f)) = Y(f) = f(n) = n`



这里不推导 Y-组合子，直接给出其定义：

```
Lambda 表达式：

λf.(λx.x x) (λx.f(x x))
更广为人知的版本：
λf.(λx.f (x x)) (λx.f (x x))

// 对于没有惰性求值的语言，直接使用该版本会栈溢出，
// 为了避免该问题，可以进行 η-变换，将 f 转换成 λx.f x 推迟求值
λf.(λx.x x) (λx.f(λv.x x v))
λf.(λx.f(λv.x x v))(λx.f(λv.x x v))
```

```javascript
// JavaScript:

f => (x => x(x))(x => f(v => x(x)(v)))
// 或
f => (x => f(v => x(x)(v)))(x => f(v => x(x)(v)))
```

```scheme
;; Scheme:

(lambda (f)
  ((lambda (x)
     (x x))
   (lambda (x)
     (f (lambda (v) ((x x) v))))))
```



Y-组合子，可以用于解决 λ 函数递归的问题。下面先看经典的阶乘函数例子。

```javascript
// JavaScript:

const factorial = n => (n === 0) ? 1 : n * factorial(n - 1)
```

```scheme
;; Scheme:

(define factorial
  (lambda (n)
    (if (= n 0)
        1
        (* n (factorial (- n 1))))))
```

观察上面的实现代码，在 `factorial` 的函数体内部，又调用了 `factorial` 自身，这是通过对名称的显式引用来做到的。



而在 λ 演算中，函数是没有名称的 λ 表达式，所以以上办法是行不通的，没办法直接做到递归。那只能从间接方式入手。



我们先写出一个假想的 λ 表达式，递归部分，我们先使用一个假设的等价的阶乘函数 `factorial` 占位：

```javascript
// JavaScript:

n => (n === 0) ? 1 : n * factorial(n - 1)
```

```scheme
;; Scheme:

(lambda (n)
  (if (= n 0)
      1
      (* n (factorial (- n 1)))))
```

问题就变成了，代码中假设的 `factorial` 部分是没有定义的，需要解决。



一个比较自然的想法是，我们可以用一个新的函数包装，将 `factorial` 以参数的形式传递进去，即：

```javascript
// JavaScript:

factorial => n => (n === 0) ? 1 : n * factorial(n - 1)
```

```scheme
;; Scheme:

(lambda (factorial)
  (lambda (n)
    (if (= n 0)
        1
        (* n (factorial (- n 1))))))
```

然后我们获得了一个会生成阶乘函数的 “元函数” (下文称 `meta_fact`），通过这样调用：

```
meta_fact(factorial)
```

即可生成一个阶乘函数，可以整理出以下等式：

```
meta_fact(factorial) = factorial
```

即，`factorial` 为 `meta_factorial` 的一个不动点。于是，我们使用 Y-组合子便可获得该最终的阶乘函数：

```javascript
const Y = f => (x => f(v => x(x)(v)))(x => f(v => x(x)(v)))
const meta_fact = (factorial => n => (n === 0) ? 1 : n * factorial(n - 1))
const factorial = Y(meta_fact)
```
