---
title: "【译】浏览器 user-agent 的历史"
description: "浏览器 user-agent 的历史。"
date: 2021-08-11T20:52:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["浏览器", "UA", "浏览器历史", "翻译"]
categories: ["编程"]
featuredImage: "featured-image.webp"
---

浏览器 user-agent 的历史。

<!--more-->

> 本文翻译自：[https://webaim.org/blog/user-agent-string-history/](https://webaim.org/blog/user-agent-string-history/)

## 为什么翻译这篇文章？

在了解浏览器引擎（注意，这里仅指渲染引擎，Browser Engine 在 [Wiki](https://en.wikipedia.org/wiki/Browser_engine) 里仅指浏览器的渲染引擎，不要与 JS 引擎混淆）的时候，我知道浏览器厂商们基本上是互相“[借鉴](https://en.wikipedia.org/wiki/Browser_engine#Notable_engines)”。例如早期 Linux 上的浏览器使用的是 KHTML 引擎，然后 Apple fork 了 KHTML 创建了 WebKit（注：WebKit 包含了 WebCore 和 JavaScriptCore，但后来随着 JS 引擎独立性越来越强，基本 WebKit 也可用来仅表示 WebCore），Google 基于 WebKit 创建了 Blink 等等。

在我的 Chrome 浏览器上查看网络请求时，我发现请求头中的 user-agent 为：_user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36_。我感到很困惑，为什么这里还有 _Mozilla_，_Safari_ 等等字样，当然最好奇的还是 _(KHTML, like Gecko)_，这是什么鬼？为什么要像 Gecko（Gecko 是 Firefox 浏览器的引擎）。

带着这样的疑问，我在 Stack Overflow 上找到了[答案](https://stackoverflow.com/questions/26112108/what-does-khtml-like-gecko-mean-in-a-user-agent-string)。并从回答中找到了关于 user-agent 历史的介绍，也就是我即将翻译的这篇文章，我觉得很有趣。

以下是正文：

## 正文

{{< image src="./mosaic.jpeg" caption="NCSA Mosaic browser" class="tzch-original-size" >}}

[NCSA Mosaic](<https://en.wikipedia.org/wiki/Mosaic_(web_browser)>) 作为早期的浏览器之一，它的 user-agent 为 _NCSA_Mosaic/2.0 (Windows 3.1)_。Mosaic 可以将图片和文字一起展示在网站上，大家对此很开心。

{{< image src="./netscape.jpeg" caption="Netscape browser" class="tzch-original-size" >}}

随后，出现了一款新的浏览器 [Mozilla](http://en.wikipedia.org/wiki/Mozilla)，Mozilla 是 "Mosaic Killer" 的缩写，Mosaic 并不觉得这有趣，因此 Mozilla 又把名字改为了 Netscape，Netscape 的 user-agent 为 _Mozilla/1.0 (Win3.1)_。更让大家开心的是，Netscape 支持 [frame](<https://en.wikipedia.org/wiki/Frame_(World_Wide_Web)#Tags_and_attributes>)，frame 变得流行起来。但是 Mosaic 浏览器不支持 frame，因此网站管理员开始使用“[user-agent 嗅探](https://en.wikipedia.org/wiki/User_agent#User_agent_sniffing)”，对于 user-agent 中包含 “Mozilla” 的浏览器发送 frames，其他的浏览器则不发送。

{{< image src="./ie.png" caption="IE browser" class="tzch-original-size" >}}

Netscape 说道，让我们嘲笑微软吧，Windows 操作系统简直就是一组没有得到良好调试的设备驱动器。微软为此感到生气，因此开发了自己的浏览器，将其称作 Internet Explorer，并希望它能成为“Netscape Killer”。IE 浏览器也支持 frame，但它毕竟不是 Mozilla，因此网站不会给他返回 frame。微软变得不耐烦了，不想等到网站管理员意识到 IE 且它支持 frame 时才向其发送 frame，因此 IE 声称它自己是 “兼容 Mozilla 的”，开始冒充 Netscape，它的 user-agent 为 _Mozilla/1.22 (compatible; MSIE 2.0; Windows 95)_，这样它就可以收到 frame 了，微软全体员工都感到高兴，但是网站管理员们则变得困惑起来。

{{< image src="./mozilla.png" caption="Mozilla browser" class="tzch-original-size" >}}

{{< image src="./firefox.jpeg" caption="Firefox browser" class="tzch-original-size" >}}

微软开始将 IE 和 Windows 捆绑销售，IE 也变得比 Netscape 更好，第一场浏览器大战打响了。随后不久，Netscape 便败下阵来，微软变得更开心了。但是 Netscape 后来又以 Mozilla 的身份重生，Mozilla 开发了 [Gecko 引擎](<https://en.wikipedia.org/wiki/Gecko_(software)>)，此时它的 user-agent 为 _Mozilla/5.0 (Windows; U; Windows NT 5.0; en-US; rv:1.1) Gecko/20020826_，Gecko 作为渲染引擎非常好用。随后 Mozilla 又更名成 Firefox，此时的 user-agent 变为 _Mozilla/5.0 (Windows; U; Windows NT 5.1; sv-SE; rv:1.7.5) Gecko/20041108 Firefox/1.0_。Firefox 在当时非常好用，其他浏览器也开始基于 Gecko 的代码打造，Gecko 变得火了起来。其中一个浏览器（Camino）的 user-agent 叫做 _Mozilla/5.0 (Macintosh; U; PPC Mac OS X Mach-O; en-US; rv:1.7.2) Gecko/20040825 Camino/0.8.1_，另一个浏览器（SeaMonkey）则自称 _Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.8.1.8) Gecko/20071008 SeaMonkey/1.0_，它们都假装自己是 Mozilla，且它们的渲染引擎都使用了 Gecko。

{{< image src="./konqueror.jpeg" caption="Konqueror browser" class="tzch-original-size" >}}

基于 Gecko 的浏览器比 IE 更好，相关的 user-agent 嗅探再次出现，基于 Gecko 的浏览器可以获取到良好的 web 代码，但是其他浏览器则不行（译者注：这里意译就是网站管理员们再次针对 Gecko 类的浏览器和非 Gecko 类的浏览器进行区分，从而发送不同的 web 代码，例如 html,js,css 代码等等）。一些 Linux 的追随者对此感到沮丧，因为他们开发了 Konqueror 浏览器，基于 KHTML 引擎，他们认为 KHTML 引擎的表现和 Gecko 一样，但是由于它们不是 “Gecko”，因此网站管理员们不会给它们返回良好的网页代码，因此 Konqueror 浏览器也开始假装自己“类似 Gecko（like Gecko）“以获得良好的网页，它的 user-agent 为 _Mozilla/5.0 (compatible; Konqueror/3.2; FreeBSD) (KHTML, like Gecko)_，造成了更多的困惑。

{{< image src="./opera.jpeg" caption="Opera browser" class="tzch-original-size" >}}

后来 Opera 浏览器出现了，它宣称：”当然，我们应该允许我们的用户选择模拟哪个浏览器“。因此 Opera 提供了一个菜单项。根据用户的选择不同，Opera 的 user-agent 要么为 _Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; en) Opera 9.51_，或者 _Mozilla/5.0 (Windows NT 6.0; U; en; rv:1.8.1) Gecko/20061208 Firefox/2.0.0 Opera 9.51_，或者 _Opera/9.51 (Windows NT 5.1; U; en)_。

{{< image src="./safari.jpeg" caption="Safari browser" class="tzch-original-size" >}}

Apple 也开发了 Safari 浏览器，Safari 使用的引擎 fork 了 KHTML 引擎的代码，并额外添加了许多功能，叫做 WebKit，为了兼容 KHTML（注：这里我翻译成兼容，意思是为了假装自己支持 KHTML，从而网站管理员能返回正确的页面代码。就像上面提到的一样，网站管理员可能会嗅探 user-agent，针对特定的一些浏览器发送不同的页面代码，因此新的浏览器需要假装自己是那些老的浏览器以便获得良好的页面代码），Safari 的 user-agent 为 _Mozilla/5.0 (Macintosh; U; PPC Mac OS X; de-de) AppleWebKit/85.7 (KHTML, like Gecko) Safari/85.5_，情况因此变得更糟了。

微软由于非常害怕 Firefox，IE 浏览器再度归来，user-agent 被改为了 _Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.0)_，这样一来，只要网站管理员们”认为“它是 Mozilla 并向它发送良好的页面代码，它就可以收到并渲染出良好的页面了。

{{< image src="./chrome.jpeg" caption="Chrome browser" class="tzch-original-size" >}}

再后来，Google 开发了 [Chrome](http://www.google.com/chrome) 浏览器，他使用的是 WebKit 引擎。和 Safari 当时为了兼容 KHTML 一样，Chrome 为了兼容 Safari，也假装自己是 Safari。因此，Chrome 使用 WebKit，它假装自己是 Safari，WebKit 又基于 KHTML，它假装自己是 KHTML，而 KHTML 则假装自己是 Gecko，最后，所有的浏览器都假装自己是 Mozilla。Chrome 的 user-agent 因此变为 _Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US) AppleWebKit/525.13 (KHTML, like Gecko) Chrome/0.2.149.27 Safari/525.13_。这样一来，user-agent 已经一团糟了，而且几乎毫无用处，每一个浏览器都假装自己是其他浏览器，令人困惑的情况比比皆是。

## 回答问题

读完了 user-agent 历史，我们再来看看并回答最开始我提到的问题。

我的 Chrome 浏览器 user-agent 为（运行 OS 为 macOS）：_user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36_。

1. 为什么包含 _Mozilla_？

   答：因为早期的 Netscape 浏览器最开始就叫做 Mozilla，其 user-agent 为 _Mozilla/1.0 (Win3.1)_。Netscape 打败了当时的对手 Mosaic 浏览器，因为其支持 frame 而 Mosaic 不支持。此时的网站管理员们开始使用 user-agent 来判断，如果包含了 _Mozilla_ 才向其发送 frame。因此后来的浏览器为了实现 frame，都假装自己是 Mozilla，也就是在 user-agent 中包含 _Mozilla_ 字样。

2. 为什么包含 _Safari_？

   答：Chrome 基于的引擎是 Apple 最初开发并用于 Safari 的 WebKit。Safari 的 user-agent 包含了 _AppleWebKit_ 和 _Safari_ 等字样。为了维持兼容性（也就是防止网站管理员嗅探了这些字样），因此包含了 _Safari_。

3. 为什么包含 _(KTML, like Gecko)_？

   答：WebKit 引擎是基于 KHTML 引擎开发的，同样是为了维持兼容性，因此加上了 _KHTML_ 字样。而早期的 KHTML 引擎只被广泛用于 Linux，而同期 Mozilla 的 Firefox 浏览器更火，特别是其 Gecko 引擎，很多浏览器都基于 Gecko 引擎，因此网站管理员对 _Gecko_ 进行了嗅探。使用基于 KHTML 浏览器的 Linux 追随者们认为 KHTML 本和 Gecko 一样优秀，因此它们加上了 _KHTML,like Gecko_ 以跳过嗅探，实现兼容。
