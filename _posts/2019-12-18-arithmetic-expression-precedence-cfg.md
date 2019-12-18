---
layout: post
title:  "算术表达式解析系列之文法规则实现优先级"
categories: JavaScript
tags: [JavaScript]
---

本篇是算术是表达式解析系列文章之一。

系列前篇：

- [算术表达式解析系列之逆波兰表示法](https://dawenci.me/posts/20191211-arithmetic-expression-shunting-yard)
- [算术表达式解析系列之优先级爬升法](https://dawenci.me/posts/20191212-arithmetic-expression-precedence-climbing)


建议先阅读上篇，本篇不重复介绍一些基本的概念。

> 本篇中，大量跟上篇相同代码实现，考虑到篇幅，这里也不再重复给出。

本篇将使用跟上一篇相似的方式，采用递归下降的方式来解析语法。但是对于优先级的处理，将直接从文法层面做处理。
这种方式，从学习和理解的角度上来讲，可能会更简单一些。不过在性能方面，会更差一些。

> 实际上，这系列三篇中介绍的方法，性能方面，是降序排列的。  


## 文法

上一篇中，用了递归下降来解析表达式的语法。

而解析语法的依据，则是我们定制的文法。这就好比如我们的自然语言，也要依照语法规则来组织单词，才能成句一样的道理。

英文有英文的语法规则，中文有中文的语法规则，同样，我们的表达式语言中，也对应着一套规则。

在本篇中，我们的表达式对应的文法大致如下（表示方法并不严谨）：

```
<表达式> = <加减表达式>
<加减表达式> = <乘除模表达式><加减表达式尾部>
<加减表达式尾部> = <加减运算符><乘除模表达式><加减表达式尾部> | ε
<加减运算符> = + | -
<乘除模表达式> = <指数表达式><乘除模表达式尾部>
<乘除模表达式尾部> = <乘除模运算符><指数表达式><乘除模表达式尾部> | ε
<乘除模运算符> = * | / | %
<指数表达式> = <取负表达式><指数表达式尾部>
<指数表达式尾部> = <指数运算符><取负表达式><指数表达式尾部> | ε
<指数运算符> = ^
<取负表达式> = <负运算符><主表达式> | <主表达式>
<负运算符> = -
<主表达式> = (<表达式>) | <数字字面量>
<数字字面量> = {1|2|3|4|5|6|7|8|9}
```

> 大体上，可以将 `=` 读作 “由...组成” ，尖括号围绕的代表某种语法成分，`|` 代表或，`ε` 代表空。

代码实现中，将会按照这个文法，从上往下调用对应的解析方法。

## 实现

先看 `Parser` 的实现代码，再解释。

> 重点关注 `parseExpr` 之后的代码，之前部分跟上篇的一致。  
> 代码中，各个子程序入口，都注释上了对应的文法规则。


```typescript
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

  // <表达式> = <加减表达式>
  parseExpr(): Ast {
    return this.parseAddExpr()
  }

  // <加减表达式> = <乘除模表达式><加减表达式尾部>
  parseAddExpr() {
    const lhs = this.parseMulExpr()
    return this.parseAddTail(lhs)
  }

  // <加减表达式尾部> = <加减运算符><乘除模表达式><加减法尾部> | ε
  parseAddTail(lhs) {
    const op = this.peek()
    // 如果是加减号开头，则说明就是加减法运算
    // 解析出右操作数，返回一个二元表达式结点
    if (this.isOperator(op, '+') || this.isOperator(op, '-')) {
      this.consume()
      const rhs = this.parseMulExpr()
      const node = Ast.createNode(AstType.BinaryExpression, {
        left: lhs,
        right: rhs,
        op: {
          kind: TokenType.Operator,
          start: op.start,
          end: op.end,
          value: op.value,
        }
      })
      // node 整体作为左操作数
      return this.parseAddTail(node)
    }
    // 否则对应空的情况，直接返回左侧结点
    return lhs
  }

  // <乘除模表达式> = <指数表达式><乘除模表达式尾部>
  parseMulExpr() {
    const lhs = this.parseExpExpr()
    return this.parseMulTail(lhs)
  }

  // <乘除模尾部> = <乘除模运算符><指数表达式><乘除模表达式尾部> | ε
  parseMulTail(lhs) {
    const op = this.peek()
    // 如果是乘除号开头，说明就是乘除法
    // 解析出右操作数，返回一个二元表达式结点
    if (this.isOperator(op, '*') || this.isOperator(op, '/')) {
      this.consume()
      const rhs = this.parseExpExpr()
      const node = Ast.createNode(AstType.BinaryExpression, {
        left: lhs,
        right: rhs,
        op: {
          kind: TokenType.Operator,
          start: op.start,
          end: op.end,
          value: op.value,
        }
      })
      // node 整体作为左操作数
      return this.parseMulTail(node)
    }
    // 否则对应空的情况，直接返回左侧结点
    return lhs
  }

  // <指数表达式> = <取负表达式><指数表达式尾部>
  parseExpExpr() {
    const lhs = this.parseNegExpr()
    return this.parseExpTail(lhs)
  }

  // <指数表达式尾部> = <指数运算符><取负表达式><指数表达式尾部> | ε
  parseExpTail(lhs) {
    const op = this.peek()
    // 如果是脱号开头，说明是指数运算
    // 解析出右操作数，返回一个二元表达式结点
    if (this.isOperator(op, '^')) {
      this.consume()
      // 注意，指数运算我们规定为右结合，所以：
      // 1. operand 可能作为 op 的右操作数
      // 2. operand 可能跟后面紧跟着出现的 `^` 右结合，然后整体作为 op 的右操作数
      const operand = this.parseNegExpr()
      // 处理右结合
      const rhs = this.parseExpTail(operand)
      return Ast.createNode(AstType.BinaryExpression, {
        left: lhs,
        right: rhs,
        op: {
          kind: TokenType.Operator,
          start: op.start,
          end: op.end,
          value: op.value,
        }
      })
    }
    // 否则对应空的情况，直接返回左侧结点
    return lhs
  }

  // <取负表达式> = <负运算符><主表达式>|<主表达式>
  parseNegExpr() {
    const op = this.peek()
    if (this.isOperator(op, '-')) {
      this.consume()
      const argument = this.parsePrimaryExpr()
      const prefix = true
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

  // <主表达式> = "("<表达式>")" | <数字>
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

从 `parseExpr()` 开始，就跟上一篇的不一样了。
对照着代码中注释的文法规则，应该很容易发现，代码实现，完全就是在翻译开头定义的那套文法。


## 优先级

仔细查看上面的代码，不难发现，我们从优先级最低的加（减）法运算开始，到优先级最高的括号和单个数字字面量结束。

下面展开分析。

1. 首先入口 `parseExpr()` -> `parseAddExpr()`，我们最开始解析的是优先级最低的加减法。
2. 在 `parseAddExpr()` 解析加减法时，左操作数会调用 `parseMulExpr()` 子程序去解析。而右操作数在 `parseAddTail(lhs)` 中，也一样的调了 `parseMulExpr()`。这意味着，我们**将整个乘（除、模）法表达式视作一个整体来作为加（减）法的一个操作数**。换句话说，乘除模运算总会优先于加减法进行组合，这就是优先级的体现。
3. 同理，用 `parseMulExpr()` 解析乘除模运算时，其的左右操作数来自于指数运算解析子程序 `parseExpExpr()`，所以，指数运算的优先级高于乘除模运算。
4. 再同理，指数运算的操作数，来自于一元取负子程序 `parseNegExpr()`，所以，一元取负的优先级，高于指数运算。
5. 括号表达式被安排在最后，因此它的优先级是高于所有的运算符的，总是被视作一个整体来解析。而数字表达式已经是原子级别了，不能继续细分，跟括号表达式一起处理即可。

## 结合性

对于左结合，比较简单，看加减乘除模运算的处理代码应该不难理解。

而右结合则比较绕。

仔细查看上面的指数运算解析子程序 `parseExpExpr()` 的实现，这里针对右结合做了特殊处理。

在遇到 `^` 运算符时，解析出右操作数 `operand`，此时，我们还不能确定 `operand` 是否应该作为该 `^` 的右操作数。考虑下面两种情况：

1. `2 ^ 3 * 4`
2. `2 ^ 3 ^ 4`

其中，情况 1，`operand` 为 `3`，右边不是新的 `^` 运算符，所以，不用考虑结合性，可以直接将 `operand` 跟前面的 `^` 结合即可。

而情况 2，则需要优先将 `3 ^ 4` 进行结合，然后将结合的结果作为整体在与前面的 `2` 结合，相当于 `2 ^ (3 ^ 4)`。并且这种情况是递归的，考虑：`2 ^ 3 ^ 4 ^ 5`。

所以，在实现上，就必须在返回 Ast 结点之前，先调用 `parseExpTail(lhs)` 尝试继续组合 `^` 后续的运算符，如果后续没有出现 `^`，则会走直接返回的分支，并没有什么副作用。由此就实现了结合性的控制。

## 延伸

本篇介绍的这种递归下降的表达式解析方式，灵活性非常高，而且比较容易理解。

对于比较复杂文法规则，甚至是上下文有关的文法，都可以用递归下降的方式来手写实现。

我们这里解析算术表达式，其实是杀鸡用牛刀了。就针对我们这里的使用场景，如果要做扩展，只要注意下优先级和结合性就可以了，基本上都可以很轻松做到。

系列文章完。
