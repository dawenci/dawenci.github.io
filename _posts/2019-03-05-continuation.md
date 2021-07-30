---
layout: post
title:  "Scheme 中的 continuation"
categories: FP
tags: [FP]
---

`continuation` 是一个非常抽象晦涩的概念，为了理解这个概念，翻阅了大量的资料。
下面记录一些对 `continuation` 的粗浅的理解。

## `continuation` 代表了程序于某一点接下来将要执行的 “后续部分”。

在 scheme 中，操作符 `all‑with‑current‑continuation`（下文开始使用缩写 `call/cc`) 提供了使用 `continuation` 的方法。`call/cc` 会捕获调用处的 `continuation`，然后将该 `continuation` 传入其参数（一个 procedure）中进行处理。

> `call/cc` 接受一个 procedure（过程/函数）作为参数，`call/cc` 调用处的 `continuation` 将作为该 procedure 的参数传入。

观察这个例子：

```scheme
(+ 1 (call/cc (lambda (k) (+ 2 (k 3)))))
```

例子中，

1. 首先，我们先把 `call/cc` 所处位置部分当作一个“空洞”
2. 然后，`call/cc` 所处位置的 `continuation`，就是 “空洞” 之外的程序后续将执行的过程

```scheme
(+ 1 空洞)
```

具体点，就是 “**一个将对该（空洞）位置做加一的过程**”，相当于：

<!-- more -->

```scheme
(lambda (hole) (+ 1 hole))
```

再来观察上述例子中 `call/cc` 的参数部分，

```scheme
(lambda (k) (+ 2 (k 3)))
```

其中形参 `k` 绑定的即为 `continuation`。执行时，该 `continuation` 会应用在 `3` 上。换句话说，`3` 充当了上述的“空洞”的值，继续走完 “下半程”（即 +1 操作）。

```scheme
(+ 1 3) 
```

结果为 `4`。

## escaping continuation

需要注意的是，上述过程中，`3` “走了不寻常的路” 穿梭到 `k` 这个 `continuation` 中去了。

而原本 `(k 3)` 处的 `continuation`（即 `(+ 2 空洞)`）将被抛弃！就是说，开头的例子中，最终的运行结果就是 `4`：

```scheme
(+ 1 (call/cc (lambda (k) (+ 2 (k 3)))))
=>  4
```

这是个很有用的特性，叫做 `escaping continuation`，可以用于退出一个计算过程，在这个例子中就是退出 `+ 2` 的运算过程。可以用下面的代码加深理解：

```scheme

;; 返回
(+ 1 (call/cc (lambda (return) (+ 2 (return 3)))))
=> 4

;; 退出
(call-with-current-continuation
   (lambda (exit)
     (for-each (lambda (x)
                 (if (negative? x)
                     (exit x)))
               '(54 0 37 -3 245 19)) #t))
=> -3
```

作为比较：

```scheme
(+ 1 (call/cc (lambda (k) (+ 2 3)))))
=> 6
```

该例子中，没有使用 `call/cc` 处的 `continuation`，因此不会有逃逸现象，`(+ 2 3)` 的值作为 `call/cc` 的返回值，然后 `+ 1`，最终结果为 `6`。

scheme 的 `continuation` 还能回到上一次离开的上下文，因为 `continuation` 里面，一直持有程序的“后续”，我们可以不限制的多次执行它们，用下面的例子来说明：

```scheme
(define r #f)

(+ 1 (call/cc (lambda (k)
                ;; 将 `continuation` 保存在 `r` 中
                (set! r k)
                (+ 2 (k 3)))))
=> 4
```

这次的例子中，我们额外使用全局变量 `r` 来记录这个 `continuation`，该例子此时跟之前一样，仍然返回运算结果 `4`。
然后，我们再通过全局变量 `r` 来再次“重放”该 `continuation`：

```scheme
(r 5)
=> 6
```

再次强调，需要注意伴随 `r` 的逃逸现象，如下面的例子：

```scheme
(+ 3 (r 5))
=>  6
```

其中 `r` 被应用在 `5` 时，`(r 5)` 本身的 `continuation` 就会被抛弃。



## 参考

[https://ds26gte.github.io/tyscheme/index-Z-H-15.html](参考)
