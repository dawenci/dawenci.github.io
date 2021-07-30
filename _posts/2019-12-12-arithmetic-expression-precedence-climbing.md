---
layout: post
title:  "算术表达式解析系列之优先级爬升法"
categories: JavaScript
tags: [JavaScript]
---

本篇是算术是表达式解析系列文章之一，上篇：[算术表达式解析系列之逆波兰表示法](https://dawenci.me/posts/20191211-arithmetic-expression-shunting-yard)

建议先阅读上篇，本篇不重复介绍一些基本的概念。

再这篇文章里面，将介绍一种完全不同的解决方案，**Precedence Climbing Method**，下称**优先级攀爬算法**。
这种算法，在手写一些表达式解析器的时候，经常会使用。

下面简单概括下实现：

1. 解析录入的原始字符串，输出 token 流
2. 逐个读取 token，根据算符不同，执行不同的操作

下面开始实现，代码使用 ts 做演示。

目标跟上一篇的相似，提供一样的运算符，不过再几个方面做了强化：

1. 数值支持更多的形式，例如： `10`, `-10`, `0.5e2` 等等等等
2. 提供完整的 Tokenize 实现，直接录入原始表达式字符串，即可运算出结果
3. 提供更友好的错误提示，包括指示出错误出现的行号列号

---

## 实现 tokenize

tokenize 是将原始字符串录入，转换成一个个 token 的过程。

转换成 token 的目的，是为了简化后续的处理步骤。

实现 tokenize 有很多方法，这里使用手工实现，遍历读取字符进行分析。

首先实现一个简单的流抽象类，用于后续读取原始输入和 tokens。

<!-- more -->

### 流

```ts
abstract class Stream<T> {
  // 当前流过的位置
  private _count = 0

  get count() {
    return this._count
  }

  set count(n: number) {
    this._count = n
  }

  /**
   * 获取缓冲数据抽象方法
   */
  abstract buffer(): any

  /**
   * 是否流干
   */
  drained(): boolean {
    return this.peek() == null
  }

  /**
   * 流到下一个元素
   */
  junk(): void {
    this.count += 1
  }

  /**
   * 流到下一个元素，返回当前元素
   */
  next(): T {
    const e = this.peek()
    if (e != null) this.junk()
    return e
  }

  /**
   * 当前元素
   */
  peek(): T {
    return this.buffer()[this.count]
  }

  /**
   * 读 N 个元素（不移动当前位置）
   * @param n
   */
  npeek(n: number): T[] {
    const count = this.count
    const list = []
    while (n--) {
      list.push(this.next())
    }
    this.count = count
    return list
  }
}
```

实现一个字符流，接受表达式源码，返回一个流对象，方便管理对字符的读取。


```ts
class CharStream extends Stream<string> {
  // 字符缓冲
  private _buffer: string
  // 当前行号
  private _row = 0
  // 当前列号
  private _col = 0

  constructor(buffer: string) {
    super()
    this._buffer = buffer || ''
  }

  get row() {
    return this._row
  }

  get col() {
    return this._col
  }

  /**
   * @override
   */
  buffer(): string {
    return this._buffer
  }

  /**
   * @override
   */
  npeek(n: number): string[] {
    const { row, col } = this
    const chars = super.npeek(n)
    this._row = row
    this._col = col
    return chars
  }

  /**
   * @override
   */
  junk(): void {
    super.junk()
    if (this.peek() === '\n') {
      this._row += 1
      this._col = 0
    }
    else {
      this._col += 1
    }
  }

  /**
   * 在当前位置开始的子串中，执行正则匹配
   */
  execRegExp(regexp: RegExp, n?: number) {
    return regexp.exec(this._buffer.substr(this.count, n))
  }
}

```

接着，根据我们的需要，定义一批 Token 的类型。
我们目前只需要三类 Token，分别为数字、操作符、分界符。

```ts
const enum TokenType {
  Number = 'Number',
  Operator = 'Operator',
  Punctuator = 'Punctuator',
}
```

确定类型之后，就可以实现一个 Token 类用来存储解析过程遇到的各种类型的单词信息。

我们计划在 Token 中存记录以下信息：

1. 类型，即 `TokenType`
2. Token 位于原始表达式输入的起始位置，用于出错时，友好地提示。
3. token 的值是什么。

根据这些要求，我们写出以下代码：

```ts
// 原始输入的表达式的位置
interface Position {
  row: number
  col: number
}

// Token 的实例化参数
interface TokenOptions {
  kind: TokenType
  start: Position
  end: Position
  value?: string
}

// 类实现
class Token {
  kind: TokenType
  start: Position
  end: Position
  value?: string
  constructor({ kind, value, start, end }: TokenOptions) {
    Object.assign(this, { kind, value, start, end })
  }
}
```

有了以上准备后，我们就可以着手开始 Token 流的实现了，使用统一的抽象方式，简化后续的处理。

```ts
// 工具函数，检查输入的字符是否数字
const isDigit(char) => /^[0-9]/.test(char)

// 类实现
class TokenStream extends Stream<Token> {
  private _buffer: any = []
  private _srcStream: CharStream

  constructor(source: string) {
    super()
    this._srcStream = new CharStream(source.trim())
  }

  /**
   * @override
   */
  buffer(): Token[] {
    return this._buffer
  }

  /**
   * @override
   */
  peek() {
    while (this.buffer().length <= this.count && !this._srcStream.drained()) {
      this.force()
    }
    return super.peek()
  }

  // 获取当前读取到的字符所处的位置
  pos() {
    return {
      row: this._srcStream.row,
      col: this._srcStream.col
    }
  }  

  // 抛出词法错误
  raiseLexicalError(message?: string) {
    const { row, col } = this.pos()
    throw new Error(`Lexical error(${row}, ${col})${message ? ': ' + message : ''}`)
  }

  // 原始输入中，往前推进 n 个字符
  srcJunk(n = 1) {
    while (n--) {
      this._srcStream.junk()
    }
  }

  // 从原始输入流中解析出下一个 token
  force() {
    if (this._srcStream.drained()) return null

    while (!this._srcStream.drained()) {
      const start = this.pos()
      const ch = this._srcStream.peek()

      // 读取到空格或换行，此处直接移除
      if (ch === '\n' || ch === ' ') {
        this.srcJunk()
        continue
      }

      // 读取到数字或小数点，说明这是一个数字 token 的开始
      if (isDigit(ch) || ch === '.') {
        const number = this.slurpNumber()
        if (!number) this.raiseLexicalError('invalid number')
        const end = this.pos()
        const token = new Token({ kind: TokenType.Number, value: number, start, end })
        this._buffer.push(token)
        return token
      }

      // 分界符，“(”，“)” 用于分组
      if (['(', ')'].includes(ch)) {
        this.srcJunk()
        const end = this.pos()
        const token = new Token({ kind: TokenType.Punctuator, value: ch, start, end })
        this._buffer.push(token)
        return token
      }

      // 二元运算符
      if (['+', '-', '*', '/', '%', '^'].includes(ch)) {
        this.srcJunk()
        const end = this.pos()
        const token = new Token({ kind: TokenType.Operator, value: ch, start, end })
        this._buffer.push(token)
        return token
      }

      this.raiseLexicalError('无效的符号')
    }
  }

  // 尝试提取数字
  maybeNumber() {
    const reg = /^\d+\.\d+(?:e[\+\-]?\d+)?|^\d+\.?(?:e[\+\-]?\d+)?|^\.\d+(?:e[\+\-]?\d+)?/i
    const matched = this._srcStream.execRegExp(reg)
    const number = matched ? matched[0] : ''
    return number
  }

  // 从原始输入流中，提取数字
  slurpNumber() {
    const number = this.maybeNumber()
    if (!number) this.raiseLexicalError()
    this.srcJunk(number.length)
    return number
  }
}
```

通过上面的实现代码，我们做到了每一次调用 `next()` 方法，就能自动读取出下一个 token。

如此一来，我们就可以将 Tokenize 的动作，整合进语法分析的过程之中，而无需做一遍词法分析的独立过程。


## 语法分析

在这之前，需要先定义好运算符的优先级和结合性。

```ts
const Precedence = Object.freeze({ L1: 1, L2: 2, L3: 3 })
const enum Associativity { Left, Right }
```

我们此处的实现，只有三个优先级。而结合性，只存在左结合和右结合之分。

然后开始定义我们希望支持的运算符，指定它们的优先级和结合性。

```ts
// 二元运算符优先级以及结合性
const BINARY_OP_INFO = Object.freeze({
  // 加减
  '+': [Precedence.L1, Associativity.Left],
  '-': [Precedence.L1, Associativity.Left],

  // 乘除
  '*': [Precedence.L2, Associativity.Left],
  '/': [Precedence.L2, Associativity.Left],
  '%': [Precedence.L2, Associativity.Left],

  // 指数运算
  '^': [Precedence.L3, Associativity.Left],
})

```

定制好运算符后，就可以准备实现语法分析类。
语法分析类，接受一个原始表达式字符串作为输入，生成对应的语法分析器对象。执行对象上的分析逻辑，即可输出该表达式的 AST。

所以，我们还得准备下，先定义好必要的 AST 结点。

首先是结点的类型定义，我们使用一个枚举来定义。

```ts
const enum AstType {
  Number = 'Number',
  BinaryExpression = 'BinaryExpr',
  UnaryExpression = 'UnaryExpr',
}
```

可能存在三种结点，分别是

1. 数字
2. 二元表达式
3. 一元表达式（负数，例如 -1）

然后实现 AST 节点类。

```ts
class Ast {
  kind: AstType
  [key: string]: any

  static createNode(kind, options) {
    return new Ast(kind, options)
  }

  constructor(kind, options) {
    this.kind = kind
    Object.assign(this, options)
  }
}
```

现在，万事俱备，不欠东风，可以开始分析语法了。
下面使用递归下降的方式手写一个简单的语法分析器。
关于二元运算符的的优先级和结合性的处理，重点关注 `parseBinaryExpr()` 方法中的注释。

```ts
class Parser {
  tokenSteam: TokenStream

  constructor(src: string) {
    this.tokenSteam = new TokenStream(src)
  }

  // 抛出语法错误
  raiseSyntaxError(start: Position, end: Position, message?: string) {
    throw new Error(`Syntax error(${start.row},${start.col}-${end.row},${end.col})${message ? ': ' + message : ''}`)
  }

  // 抛出语义错误
  raiseSemanticError(start: Position, end: Position, message?: string) {
    throw new Error(`Semantic error(${start.row},${start.col}-${end.row},${end.col})${message ? ': ' + message : ''}`)
  }

  // 读取当前的 Token
  peek() {
    return this.tokenSteam.peek()
  }

  // 读取下一个 Token
  next() {
    return this.tokenSteam.next()
  }

  // 消耗一个 Token
  consume() {
    this.tokenSteam.junk()
  }

  // 检查目标 Token 是否指定的运算符
  isOperator(token?: Token, value?: string) {
    if (!token || !value) return false
    return token.kind === TokenType.Operator && token.value === value
  }

  // 检查指定 Token 是否指定的分解符
  isPunctuator(token?: Token, value?: string) {
    if (!token || !value) return false
    return token.kind === TokenType.Punctuator && token.value === value
  }

  // 解析表达式的入口
  parseExpr(): Ast {
    const expression = this.parseBinaryExpr(Precedence.L1)
    return expression
  }

  // 解析二元表达式
  parseBinaryExpr(minPrec): Ast {
    let lhs = this.parseUnaryExpr()

    // 此处使用优先级攀爬（Precedence climbing）算法处理二元表达式
    while (true) {
      // 读取当前 Token，假设是运算符
      let op = this.peek()

      // 若读取到的 Token 为空，或者运算符定义中找不到，或新运算符的优先级达不到要求的最低优先级
      // 则不再与后续的运算符结合，结束本次循环，直接返回当前已经组装的 ast 子树
      if (!op || !BINARY_OP_INFO[op.value] || BINARY_OP_INFO[op.value][0] < minPrec) {
        return lhs
      }

      // 获取运算符的优先级和结合性，算出递归调用时传入的最小优先级数值
      // 此处是关键点，通过指定最小优先级，解决不同优先级运算符的结合情况
      // 而右结合的情况，也通过 `+1` 巧妙地转换成优先级一样的处理逻辑了
      const [prec, assoc] = BINARY_OP_INFO[op.value]
      const nextMinPrec = assoc === Associativity.Left ? prec + 1 : prec

      // 消耗掉当前的 token，并准备下一个 token，用于下轮递归
      this.consume()
      const rhs = this.parseBinaryExpr(nextMinPrec, isArgComma)

      // 使用新值更新 lhs
      lhs = Ast.createNode(AstType.BinaryExpression, {
        op: {
          kind: TokenType.Operator,
          start: op.start,
          end: op.end,
          value: op.value,
        },
        left: lhs,
        right: rhs
      })
    }
  }

  // 处理一元表达式，即处理一元取负运算符
  parseUnaryExpr(): Ast {
    const op = this.peek()
    if (this.isOperator(op, '-')) {
      this.consume()
      const argument = this.parsePrimaryExpr()
      return Ast.createNode(AstType.UnaryExpression, {
        prefix: true,
        op: {
          kind: TokenType.Operator,
          start: op.start,
          end: op.end,
          value: op.value,
        },
        argument
      })
    }
    return this.parsePrimaryExpr()
  }

  // 处理主表达式，包括数字这种不可再细分的语法成分。而括号分组也在此处视作一个整体对待。
  parsePrimaryExpr(): Ast {
    const token = this.peek()

    // 括号表达式
    if (this.isPunctuator(token, '(')) {
      this.consume()
      // 括号里面可能还是一个表达式
      const expression = this.parseExpr()

      // 必须存在关闭的括号，否则就是括号不匹配的语法错误
      const closingParen = this.peek()
      if (!this.isPunctuator(closingParen, ')')) {
        this.raiseSyntaxError(closingParen.start, closingParen.end, `缺失符号 ')'`)
      }
      this.consume()

      return expression
    }

    // 数值字面量表达式
    if (token.kind === TokenType.Number) {
      this.consume()
      return Ast.createNode(AstType.Number, {
        value: token.value,
        start: token.start,
        end: token.end
      })
    }

    // 全部不匹配，报告语法错误
    this.raiseSyntaxError(token.start, token.end, `无效的符号 ${token.value}`)
  }
}
```

上面 100 多行的代码，即可实现我们这个简单的解析器的工作。
使用方式如下：

```ts
const parser = new Parser('1+1')
const ast = parser.parseExpr()
// ast 即为目标抽象语法树，数据结构大概如下：
//
// {
//   "kind": "BinaryExpr",
//   "op": {
//     "kind": "Operator",
//     "start": {
//       "row": 0,
//       "col": 1
//     },
//     "end": {
//       "row": 0,
//       "col": 2
//     },
//     "value": "+"
//   },
//   "left": {
//     "kind": "Number",
//     "value": "1",
//     "start": {
//       "row": 0,
//       "col": 0
//     },
//     "end": {
//       "row": 0,
//       "col": 1
//     }
//   },
//   "right": {
//     "kind": "Number",
//     "value": "1",
//     "start": {
//       "row": 0,
//       "col": 2
//     },
//     "end": {
//       "row": 0,
//       "col": 3
//     }
//   }
//  }
//
```

实现了 AST 输出之后，即可实现计算逻辑，这部分就非常简单了，就是树遍历的过程。

> 注，在我们的例子中 AST 的生成其实不是必须的，在分析的过程中，已经可以执行相应的计算语义了。

下面给出一个简单的计算方法实现。


先定义一个运行时库，包含各种运算符的计算逻辑

```ts
const runtime = {
  operators: {
    '+': (a: number, b: number) => a + b,
    '-': (a: number, b: number) => a - b,
    '*': (a: number, b: number) => a * b,
    '/': (a: number, b: number) => a / b,
    '%': (a: number, b: number) => a % b,
    '^': (a: number, b: number) => Math.pow(a, b),
    'neg': (a: number) => -a,
  }
}
```

接着实现树遍历计算即可。

```ts
function evaluateAst(ast: Ast) {
  switch (ast.kind) {
    // 二元表达式
    case AstType.BinaryExpression: {
      // 提取运行时中的操作方法
      const calc = runtime.operators[ast.op.value]
      // 递归计算出左操作数
      const lhs = evaluateAst(ast.left)
      // 递归计算出右操作数
      const rhs = evaluateAst(ast.right)
      // 返回计算结果
      return calc(lhs, rhs)
    }

    // 一元表达式
    case AstType.UnaryExpression: {
      const arg = evaluateAst(ast.argument)
      return runtime.operators.neg(arg)
    }

    // 数字字面量，直接字符串转数字即可
    case AstType.Number: {
      return JSON.parse(ast.value)
    }
  }
}
```

综合上面所有的实现，最终可以封装一个计算 API，提供给外部使用。

```ts
function evaluate(src: string) {
  return evaluateAst(new Parser(src).parseExpr())
}
```

调用

```ts
evaluate('(10 + 15 - 20) * 30 / 40 ^ 2')
// 输出：0.09375
```

跟上一篇的方法，完全一样的输出。


## 下篇预告

算术表达式解析系列之文法规则实现优先级
