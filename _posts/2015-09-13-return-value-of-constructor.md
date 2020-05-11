---
layout: post
title:  "构造函数的返回值"
categories: JavaScript
tags: [JavaScript, OOP]
---


构造函数不需要显式 `return`，如果指定了返回值，则根据返回值的类型有不同的结果。


### 代码：

```js
function PrimitiveUndefined() { }
function PrimitiveNull() { return null }
function PrimitiveBoolean() { return true }
function PrimitiveNumber() { return 123 }
function PrimitiveString() { return 'string' }
function PrimitiveSymbol() { return Symbol() }
function BooleanObject() { return new Boolean(true) }
function NumberObject() { return new Number(123) }
function StringObject() { return new String('string') }
function DateObject() { return new Date() }
function RegExpObject() { return /test/ }
function FunctionObject() { return function(){} }
function IteralObject() { return {} }
function IteralArray() { return [] }

const FUNCTIONS = [
    'PrimitiveUndefined',
    'PrimitiveNull',
    'PrimitiveBoolean',
    'PrimitiveNumber',
    'PrimitiveString',
    'PrimitiveSymbol',
    'BooleanObject',
    'NumberObject',
    'StringObject',
    'DateObject',
    'RegExpObject',
    'FunctionObject',
    'IteralObject',
    'IteralArray'
]

FUNCTIONS.forEach(func => {
    let obj = eval('new ' + func)
    let type = obj.constructor.name

    console.log('%c%s %c: %c%s',
        'color:#607ed8', func,
        'color:#888',
        'color:#d87860', type)
})

```

测试结果打印：

```
PrimitiveUndefined : PrimitiveUndefined
PrimitiveNull : PrimitiveNull
PrimitiveBoolean : PrimitiveBoolean
PrimitiveNumber : PrimitiveNumber
PrimitiveString : PrimitiveString
PrimitiveSymbol : PrimitiveSymbol
BooleanObject : Boolean
NumberObject : Number
StringObject : String
DateObject : Date
RegExpObject : RegExp
FunctionObject : Function
IteralObject : Object
IteralArray : Array
```

### 总结

* 若手动 return 一个原始类型的值 (primitive value)，则 return 语句不会起作用，仍然返回构造函数隐式创建的对象。
* 若手动 return 一个对象类型的值，则会丢弃构造函数隐式创建的对象，返回 return 语句指定的对象。


