---
layout: post
title:  "Vue 的一些笔记"
categories: JavaScript
tags: [JavaScript, Vue]
---

## Vue 实例化过程关于数据方面的处理

```js
var vm = new Vue({ data: ..., ... })
```

1. 使用构造函数的数据初始化实例的 `_data`属性(即`vm._data`)

2. 将`_data`属性，使用getter、setter代理到vue实例`vm`上
  即访问、修改vue的属性的时候，只是简单地通过getter、setter访问`vm._data`属性。

3. 观察`vm._data`属性（这部分功能交由Observer类进行）
  * 生成一个Observer对象obs，该对象持有`vm._data`(`obs.value === vm._data`)， 通过递归，将`vm._data`的数据属性，全部用`Object.defineProperty`转换成存取器属性
  * 针对每个转换后的存取器属性，会实例化一个依赖管理器`dep`(Dep类型)对象来维护依赖该属性的观察者(Watcher类型)列表`dep.subs`。
  * 当某个属性的值被修改时，obs的setter中，会递归地观察新的属性值，并最终让该属性的依赖管理器dep去通知所有watcher（订阅者）执行其update方法(`dep.notify()`，内部遍历watcher逐个执行`watcher.update()`)
  * 每当属性值被读取，会在getter中，检测`Dep.target`(Dep类的静态属性)是否有值，有值则是一个Watcher对值的读取（Watcher类的getter中，读取属性的时候，会临时设置`Dep.target`为watcher实例自身），这时候把该watcher加入当前属性的依赖管理器dep里面，这样该属性值下次改变，该watcher就能获得通知并update

4. vue 实例中，会暴露一个`$watch(expOrFn, cb)`接口，主要在指令中使用订阅者，调用该接口，会创建一个Watcher类型的对象，watcher对象创建过程，会读取`vm._data`的对应属性，从而触发上述的obs的getter，进而自动将本watcher加入对应属性的依赖管理器`dep`里，以便在属性下次变化时获得通知有机会执行update方法。


总的来讲：

### Observer类
将数据属性转换成存取器属性，并：
  * 在每个属性的getter里面，提供值之前，分析是否是新的watcher，若是，则将其加入对应属性的依赖管理器
  * 在每个属性的setter里面，修改值之后，让依赖管理器去调用所有watcher的update方法

### Dep类
用于管理某个data的属性的所有订阅者，拥有notify方法，该方法在关联的data属性发生变化时被调用，内部会执行所有订阅者的update方法

### Watcher类
用于订阅某个data属性，当属性变化时，执行实例化时构造器中传入的回调函数

### Vue类
* 构造时，自动使用Observer类来转换、维护数据
* 暴露一个$watch方法，该方法可以生成一个watcher用于订阅data数据


---

## 模版编译

编译元素 compileElement(el, options)
  * 检查el的attributes，
  * 将attributes转成数组 attrs，
  * 调用compileDirectives(attrs, options)进行指令编译，返回一个`nodeLinkFn`，这是一个链接函数，作用是绑定指令和Vue实例
  * 将`nodeLinkFn`作为最终结果返回


编译指令 compileDirectives(attrs, options)
  * 解析每个attribute，提取修改器(modifier)、过渡效果(transiton)、动态绑定(v-bind、:)，普通指令(v-text、v-ref:xxx、v-el:xxx)等等信息
  * 每个成功解析出来的结果，就是一个指令描述(directive descriptor)，将所有解析结果推入一个数组待用
    ```js
    // 指令描述数据结构
    {
      name: 'text', // 指令名称
      attr:'v-text', // 指令原始名
      raw:'值内容', // 指令原始值
      def: { // 指令定义
        bind(){},
        update() {}
      },
      arg: arg, // 参数，`v-ref:xxx`中`xxx`部分
      modifier: modifier, // 修改器
      expression: parsed && parsed.expression, // 指令原始值中的表达式部分
      filters: parsed && parsed.filters, // 指令原始值中的过滤器部分
      interp: interpTokens,
      hasOneTime: hasOneTimeToken
    }
    ```
  * 指令描述数组作为参数，调用`makeNodeLinkFn(directiveDiscriptors)`，该函数返回一个新函数`nodeLinkFn(vm, el, host, scope, frag)`
    nodeLinkFn作为闭包可以访问makeNodeLinkFn的参数directiveDiscriptors，nodeLinkFn执行时为每个directive discriptor逐一调用vm._bindDir方法
  * 将nodeLinkFn作为结果返回

Vue.prototype._bindDir(descriptor, node, host, scope, frag)
  * 使用 `new Directive(descriptor, this, node, host, scope, frag)` 得到一个Directive实例。
  * 将Directive实例push入Vue实例的`_directive`数组属性里



