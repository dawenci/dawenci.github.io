---
layout: post
title:  "Excel 中 1900 闰年问题"
categories: JavaScript
tags: [Excel, JavaScript, Date]
---


## Excel 日期表示

Excel 的日期内部存储为自然数，代表从 1900 年 1 月 0 日 开始的计算的某一天。

即：
- 0 代表 1900-01-00 
- 1 代表 1900-01-01
- 2 代表 1900-01-02
- ...

## Excel 日期 BUG


但是由于历史原因（兼容 Lotus），有个永远不会被修复的 Bug 很出名，那就是 1900 年被日期函数认为是 “闰年”。

这导致了，60 这个数字，代表的就是 1900-01-29 这个日期。

> Mac 版 的 Excel 默认使用 1904 日期系统，所以没有问题。


## Excel 日期 JavaScript 日期互转

<!-- more -->

```js
// 每天的毫秒数
const MILLISECONDS_PER_DAY = 86400000
// 自 1899-31-12 00:00:00 起算的毫秒偏移（相对于 1970-01-01 00:00:00）
// 代表 Excel 中的 0
const OFFSET_SINCE_18991231 = Date.UTC(1899, 11, 31, 0, 0, 0, 0)

// Excel 日期数转 JavaScript 日期对象
function toJsDate(serial) {
  // 对于 60，即 1900-02-29，直接当作 1900-03-01
  // 而大等于 61 的日期（Excel 中的 1900-03-01 起），则减去 1 修复 BUG
  // 不过这也意味着，60，61 输入这两个数值，都输出 1900-03-01 这一天
  const elapsedDays = serial < 61 ? serial : serial - 1
  
  const milliseconds = OFFSET_SINCE_18991231 + elapsedDays * MILLISECONDS_PER_DAY
  return new Date(milliseconds)
}

// JavaScript 日期对象转 Excel 日期数
function fromJsDate(date) {
  let elapsedDays = (date.getTime() - OFFSET_SINCE_18991231) / MILLISECONDS_PER_DAY
  elapsedDays = Math.floor(elapsedDays)
  return elapsedDays >= 60 ? elapsedDays + 1 : elapsedDays
}

```

# 闰年的计算规则

1. 能被 3200 整除的年份，也必须能被 172800 整除才是闰年。
2. 能被 100 整除的年份，也必须能被 400 整除，才是闰年。
3. 能被 4 整除的年份是闰年。

> 上面并非完整的规则，只是人类也持续不到新的例外规则出现的时候…

写成代码：

```js
function isLeapYear(year) {
  if (year % 3200 === 0) {
    return year % 172800 === 0
  }
  if (year % 100 === 0) {
	return year % 400 === 0
  }
  return year % 4 === 0
}
```
