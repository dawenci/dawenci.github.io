---
layout: post
title:  "离线应用与 manifest.appcache"
categories: Web
tags: [JavaScript, offline, manifest]
---

manifest 文件是一个文本文件，用来告诉浏览器缓存资源的规则，可以用来构建一个离线可用的应用。

> manifest 文件的 MIME 类型是 `text/cache-manifest` 

manifest 文件使用时只需简单地在 html 文件中引入。具体方法是，在 `<html>` 标签中，增加一个 `manifest` 属性，该属性引用一个 `manifest.appcache` 文件来描述资源的缓存情况。


代码如下：

```html
<!DOCTYPE html>
<html lang="en" manifest="manifest.appcache">
<head>
  <meta charset="UTF-8">
  <title>Document</title>
</head>
<body>
</body>
</html>
```


## 浏览器兼容性

在使用前，我们都关心的兼容性…

* IE 10+
* Chrome 4+
* Safari 4+
* Firefox 3.5+
* Opera 11.5+
* Safari (iOS)
* Android Browser
* Edge

现代浏览器都有支持，尽管实现上据说都有坑。

> manifest 已从标准中移除，将来浏览器随时可能会枪毙该特性…  
> 然而，后继者 service workers 还未获得广泛的支持。

<!-- more -->

## manifest.appcache 语法

### 内容结构

以 `CACHE MANIFEST` 作为第一行，告诉浏览器这是个 manifest 文件，以 `#` 开头的是注释，以 `CACHE`, `NETWORK`, `FALLBACK` 关键字将文件划分为三种作用不同的章节段落，未使用上述3个关键字时，默认罗列的都是需要缓存的资源。

#### `CACHE`: 
表示后续罗列的是需要缓存的内容

#### `NETWORK`:
表示后续罗列的是需要在线获取的内容

#### `FALLBACK`:
表示后续罗列的是离线用户访问一个没有缓存的资源时的备选资源，如：

```
FALLBACK
# 离线时，若a.png没有缓存，则使用b.png代替
/a.png /b.png
```
还能这样：
```
FALLBACK
# 离线时，访问所有没缓存的资源都提供offline.html
/ /offline.html
```

> 引入 manifest 文件的 html 文件自身默认也会加入缓存 ？


### 文件例子

```
CACHE MANIFEST 

# 下列资源会缓存
/style.css
http://example.com/example.jpg

# 下面是备选资源描述
FALLBACK:
/ /offline.html

# 下列资源需要在线获取
NETWORK:
*
```


## 缓存细节

* 是否更新资源，取决于 manifest 文件本身是否有更新，仅仅更新资源文件而没有更新 manifest 文件，是不会重新下载新资源的。因此，对于 manifest 文件，最好不要设置缓存。为了更新 manifest 文件本身，可以增加一个注释行放版本信息（时间戳之类的）

* 资源更新是一次性完整下载，如果有某些文件没有下载成功，则更新失败，所有资源还会继续使用旧版本。


## 相关 API

### applicationCache

要了解缓存的当前状态，可以通过检查 `applicationCache.status` 来知晓。

一共有以下几种状态：

* 0: 无缓存
* 1: 闲置，应用缓存未得到更新
* 2: 检查中，正在下载描述文件并检查更新
* 3: 下载中，下载描述文件指定的资源中
* 4: 更新完成，可以 swapCache() 使用
* 5: 废弃，描述文件不存在了，无法继续访问缓存

缓存状态变化过程，也有诸多的事件触发：

* `checking`: 查找更新时触发
* `error`: 检查更新或下载资源出错时触发
* `noupdate`: 检查描述文件后，确认描述文件没更新后触发
* `downloading`: 开始下载缓存资源时触发
* `progress`: 下载缓存资源过程不断触发
* `updateready`: 新缓存下载完毕可 swapCache 时触发
* `cached`: 缓存完整可用时触发

操作缓存资源的方法

#### update() 方法

调用该方法，会检查描述文件是否更新 ( 触发 `checking` )，然后按照页面刚加载时的流程执行后续操作。一旦 `updateready` 被触发，则新版本缓存已可用，可以调用 `swapCache()` 方法启用新缓存。一旦 `cached` 被触发，则缓存已完整可用，update 流程也完毕了。

#### swapCache() 方法

一旦新缓存可用，可以使用该方法启用。一般用在 `updateready` 事件触发后。

### windowz.navigator.onLine

跟离线缓存打交道，自然需要能确认网络的在线状态， HTML 5 的 `windowz.navigator.onLine` 即提供了在线状态信息。检查该属性的值，`true` 即在线， `false` 即离线。

