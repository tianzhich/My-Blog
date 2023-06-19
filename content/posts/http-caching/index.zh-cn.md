---
title: "聊一聊HTTP缓存"
description: "HTTP 缓存作为 Web 重要的一环，理解并合理使用它能够提高 Web 性能。"
date: 2018-09-16T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["浏览器", "缓存", "计算机网络"]
categories: ["编程"]
featuredImage: "featured-image.webp"
---

HTTP 缓存作为 Web 重要的一环，理解并合理使用它能够提高 Web 性能。

<!--more-->

## 前言

- 什么是 HTTP 缓存

- 为什么需要 HTTP 缓存，因为使用 HTTP 缓存可以减少请求响应时间和避免网络时延带来的等待时间

- 如何合理使用缓存，设置不同文件的缓存时间

## 缓存的种类

缓存分为两种，*Private cache*和*Shared cache*

- *Private cache*是针对单个用户而言，一个例子就是自己的浏览器缓存，使用这些缓存实现了浏览器的前进/后退，而无需再次请求资源

- *Shared Cache*针对多个用户，比如代理服务器，一个 ISP (Internet Service Provider)或者公司都有代理服务器，用来缓存一些资源，供所有员工访问，而不必再向互联网发出请求，避免网络延迟

![](http-cache-type.png)

## 缓存控制

要清楚缓存控制，这里要先分清两个概念，一个叫做缓存存储（Cache Storage），一个叫做缓存（Caching）

其实在中文里面，可能直接都说成缓存，但是这样不助于理解下面缓存控制的前两种情况。其实缓存存储是一个实实在在的东西，而缓存只是它的一个动作

后来我又发现国人有**强缓存**和**协商缓存**的说法，强缓存就是指*Cache Storage*会直接返回 Cache 的内容，协商缓存是指*Cache Storage*向原始服务器进行验证，如果原始服务器返回*304 Not Modified*，*Cache Storage*才返回 Cache 的内容

![](chrome-memeory-cache-disk-cache.png "淘宝网首页, Chrome Dev Tools 下观察到的协商缓存(304)和强缓存 (from xxx cache)")

![](HTTPStaleness.png "强缓存和协商缓存验证资源是否过期")

上面一幅图中，*304 Not Modified*表示使用了协商缓存，而*from memory cache*和*from disk cache*则直接使用强缓存

> Chrome 在某个版本之后将以前的*from cache*变成了*from disk cache*和*from memory cache*，就跟名字一样，缓存在磁盘中的内容即使在关闭了当前 tab 之后还能使用，但是缓存到内存的内容，当关闭 tab 或者浏览器，或者浏览器崩溃后，就消失了。但是*memory cache*加载速度更快，从上图也能看出

关于强缓存和协商缓存，我没发现这两个词的英文出处，但是这两个词理解起来更形象，所以下面的缓存控制情况我也会用到这两个词

### 1. 不使用**缓存存储**

可以理解为每次的请求都不经过 Cache Storage，直接发给服务器，返回新的对象

```html
Cache-Control: no-store Cache-Control: no-cache, no-store, must-revalidate
```

### 2. 不直接使用**缓存内容**

Cache Storage 会缓存内容，但是每次需要向服务器验证资源是否过期，没过期才能使用缓存，这就是*304 Not Modified*的情况，也就是我们所说的**协商缓存**

```html
Cache-Control: no-cache
```

### 3. 公共缓存和私有缓存

例如告诉资源只能被*Private Browser Cache*缓存起来还是同时也能被*Public Proxy Server*缓存起来

```html
Cache-Control: private Cache-Control: public
```

### 4. 强缓存过期控制

`Cache-Control`优先级高于`Expired`，前者是相对时间，后者是绝对时间；前者为*General Header*的字段，后者为*Response Header*的字段

```html
Cache-Control: max-age=31536000 Expires: Wed, 21 Oct 2015 07:28:00 GMT
```

### 5. 资源验证

```html
Cache-Control: must-revalidate
```

## 缓存验证

缓存验证发生在用户刷新，或者响应头中包含`Cache-Control: must-revalidate`的情况

缓存验证有两种形式，都可以用来验证文档是否过期

- 文档的最后修改日期，响应头*Last-Modified*字段，请求头*If-Modified-Since*字段请求验证
- 对文档当前版本的唯一标识，叫做*entity tag*，响应头*ETag*字段，通常使用 MD5 Hash 实现，请求头*If-None-Match*字段请求验证

同时，缓存验证有两种验证类型

- 强验证（strong validator），文档对比一个字节一个字节地进行，严格对比
- 弱验证（weak validator），文档只有语义不同才认为是改变，例如一个 HTML 文档也许只是里面的广告发生改变，或者是页脚的日期发生改变，弱验证会认为它们还是相同的

**HTTP 默认使用的是强验证**

需要注意的是，一般情况下，*Last-Modified*字段只能作为弱验证，因为它的最小单位是秒，一秒内文档可能改变 2 次，无法做到标识每一次改变，如果使用它做强验证，则需要明确知道不会发生这种情况。当然，还有一些其他的限制，具体的限制可以看我下面的第 4 条 RFC 文档参考，这里就不细说了

而*ETag*默认作为强验证，如果需要用它实现弱验证比较困难，因为需要判断文档中不同元素的语义，确保语义变化时*ETag*的值才发生变化，而不是每次检测到字节变化时就生成一个新的值

缓存验证这部分建议有能力的同学直接查看第 4 条 RFC 参考

## 条件缓存（响应头增加*Vary*字段）

该字段指出了筛选条件，如果下一次到达*Cache Storage*的请求中的该条件与缓存时的该条件的值不一样，则重新向真实 server 发请求，否则使用缓存，看下面这幅图

![](HTTPVary.png "在响应头使用*Vary*字段条件验证*Content-Encoding")

*Vary*字段通常会对*User-Agent*进行验证，避免将移动端缓存的文档直接发给桌面端请求的用户，反之亦然

```html
Vary: User-Agent
```

## 参考

1. https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching
2. https://stackoverflow.com/questions/44596937/chrome-memory-cache-vs-disk-cache?rq=1
3. https://developer.mozilla.org/en-US/docs/Web/HTTP/Conditional_requests
4. [Weak and Strong Validators](https://www.freesoft.org/CIE/RFC/2068/138.htm)
