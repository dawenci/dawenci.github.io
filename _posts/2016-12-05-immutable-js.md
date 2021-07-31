---
layout: post
title:  "Immutable.js"
categories: JavaScript
tags: [JavaScript, FP, immutable]
---

网络上找到的一张原理图：

![](../images/2016-12-05-immutable-js.assets/1.gif)

1. 对 Immutable 对象的任何修改或添加删除操作都会返回一个新的 Immutable 对象。
2. 如果对象树中一个节点发生变化，只修改这个节点和受它影响的父节点，其它节点则进行共享。
3. Immutable 实现的原理是 Persistent Data Structure（持久性数据结构）
4. Immutable 使用了 Structural Sharing 来节省内存、提升效率（深拷贝效率低，内存占用大

全文完
