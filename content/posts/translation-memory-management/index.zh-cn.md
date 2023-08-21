---
title: "【译】内存管理（Memory Management）"
date: 2021-02-08T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["JavaScript", "MDN", "翻译", "内存", "内存管理", "垃圾回收"]
categories: ["编程"]
series: ["mdn-advance-topics"]
featuredImage: "featured-image.webp"
---

内存管理（Memory Management）。

<!--more-->

{{< admonition abstract "写在前面" >}}
原文来自 [MDN JavaScript 主题的高阶教程](https://developer.mozilla.org/en-US/docs/Web/JavaScript#advanced)部分，一共 5 篇，分别涉及继承和原型链、严格模式、类型数组、内存管理、并发模型和事件循环。本篇是第 4 篇，关于[内存管理](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)。

**2023-06-23 更新：** 今天发现，MDN 原文结构进行了调整，原来[高级教程](https://developer.mozilla.org/en-US/docs/Web/JavaScript#advanced)的 5 篇文章只剩下了 3 篇，分别是继承和原型链、内存管理、并发模型和事件循环，其中，严格模式被转移到了 [References/Misc](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference#additional_reference_pages) 下，类型数组被转移到了 [JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide#typed_arrays) 下。
{{< /admonition >}}

一些底层语言例如 C，拥有手动的内存管理指令，如[`malloc()`](https://pubs.opengroup.org/onlinepubs/009695399/functions/malloc.html)和[`free()`](https://en.wikipedia.org/wiki/C_dynamic_memory_allocation#Overview_of_functions)。相比之下，JavaScript 在对象创建时自动分配内存并且在它们不再使用时释放内存（垃圾回收机制（_garbage collection_））。这种机制会使人困惑：它让开发者觉得可以不必担心内存管理。

## 内存生命周期

任何编程语言的内存生命周期都几乎相同：

1. 分配所需内存
2. 使用分配的内存（读和写）
3. 不再需要时释放该部分内存

在所有编程语言中，第二点是明确的。而第一点和第三点，在底层语言中也是明确的，不过在高阶语言例如 JavaScript 中，多数是隐式的。

### 在 JavaScript 中分配内存

#### 初始化值

为了使开发者无需关注内存分配，初始化变量时，JavaScript 会自动分配一片内存。

```js
var n = 123; // 为一个数字分配内存
var s = "azerty"; // 为一个字符串分配内存

var o = {
  a: 1,
  b: null,
}; // 为一个对象和其属性分配内存

// (类对象) 为数组和其包含的值分配内存
var a = [1, null, "abra"];

function f(a) {
  return a + 2;
} // 为一个函数分配内存 (函数是一个可调用的对象)

// 为函数表达式分配内存（函数表达式也是对象）
someElement.addEventListener(
  "click",
  function () {
    someElement.style.backgroundColor = "blue";
  },
  false
);
```

#### 通过函数调用分配内存

一些函数调用涉及到对象内存分配。

```js
var d = new Date(); // 为一个日期对象分配内存

var e = document.createElement("div"); // 为一个DOM元素对象分配内存
```

一些方法调用涉及到为原始值或对象分配内存。

```js
var s = "azerty";
var s2 = s.substr(0, 3); // s2 是一个新字符串
// 由于字符串 s 是不可变的值,
// JavaScript 不会继续为 s2 分配内存,
// 它仅仅保存下标 0-3 以获取子串.

var a = ["ouais ouais", "nan nan"];
var a2 = ["generation", "nan nan"];
var a3 = a.concat(a2);
// 拥有4个元素的新数组 a3 由 a 和 a2 合并而成
```

### 使用已分配内存的变量

使用变量意味着对分配的内存进行读和写。例如读取一个变量（或者对象属性）的值，为其赋值，或者是函数调用时传入参数。

### 当内存不再使用时对其进行释放

大部分内存管理的问题都发生在这个阶段。而其中最困难的就是判断何时不再需要这片内存。

低层次的语言要求开发者在程序中自行判断，并进行内存释放。

一些高阶语言，例如 JavaScript，利用了一种自动内存管理的形式，称作[垃圾回收机制（garbage collection）](<https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)>)（GC）。垃圾回收器的目的是监测内存分配，并确定何时该块内存不再需要，然后回收它。这个自动过程是一个近似过程，因为一块特定内存是否仍然需要是[不确定的（undecidable）](https://en.wikipedia.org/wiki/Decidability_%28logic%29)

## 垃圾回收机制

如上所述，一块已分配内存是否“不再需要”是不确定的。因此，垃圾回收器只能限制性地解决一般性问题。这个部分将会解释几个概念，它们有助于理解主要的垃圾回收算法以及它们各自的局限性。

### 引用（References）

垃圾回收算法主要依靠的一个概念叫做*引用（reference）*。在内存管理中，一个对象是否引用了另一个对象，取决于前一个对象是否访问了另一个（显式或隐式访问）。例如，一个 JavaScript 对象引用了它的[原型对象（prototype）](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain)（隐式引用）和它自身的属性（显示引用）。

在这里，“对象”的概念不仅指 JavaScript 对象，也包含了函数作用域（或全局作用域）。

### 基于引用计数的垃圾回收机制

这是最初级的垃圾回收算法。这种算法将判定“该对象是否仍然需要”简化为“是否仍有其他对象引用了该对象”。一个对象被称为“垃圾”，或者说可回收的标准是对它的引用数为 0。

#### 举例

```js
var x = {
  a: {
    b: 2,
  },
};
// 创建了两个对象。其中一个被另一个通过自身属性引用
// 另一个则通过分配给变量 'x' 被引用
// 显然，这俩都无法回收

var y = x; // 变量 'y' 是第二个对象的第二次引用

x = 1; // 现在，被变量 'x' 引用的对象只被变量 'y' 引用了

var z = y.a; // 变量 'z' 引用了 属性 'a' 引用的对象
//   这个对象现在拥有两处引用：一个是属性 'a'，
//   另一个是变量 'z'

y = "mozilla"; //  最初被变量 'x' 引用的对象现在没有了引用，因此可以进行回收
//   然而其属性 'a' 对应的对象仍然被变量 'z' 引用，
//   因此该对象暂时无法回收

z = null; // 最初变量 'x' 引用的对象属性 'a' 对应的对象没有了引用
//   可以被回收
```

#### 限制：循环引用

如果遇到循环引用，上述的回收机制会受限。下面例子中，创建了两个对象，他们同时被各自的属性引用，造成了循环。函数调用完成后，作用域消失，它们本应该被回收。然而引用计数算法不会认为他们可被回收，因为他们都至少存在一个引用。引用计数通常会导致内存泄漏。

```js
function f() {
  var x = {};
  var y = {};
  x.a = y; // x 引用了 y
  y.a = x; // y 也引用了 x

  return "azerty";
}

f();
```

#### 真实案例

IE6 和 IE7 在 DOM 对象中使用了引用计数策略的垃圾回收器，循环引用是一个导致内存泄漏的常见错误：

```js
var div;
window.onload = function () {
  div = document.getElementById("myDivElement");
  div.circularReference = div;
  div.lotsOfData = new Array(10000).join("*");
};
```

上面的例子中，名为"myDivElement"的 DOM 元素通过属性”circularReference“循环引用了自身。如果该属性没有被移除或置为`null`，基于引用计数的垃圾回收器将会认为该 DOM 对象至少存在一个引用，即使 DOM 节点从 DOM 树中移除，仍会占用内存。如果 DOM 元素含有大量数据（上面例子中的`lotsOfData`属性），那么存储这部分数据的内存将不会被释放，可能会出现内存有关的问题，例如浏览器变得十分卡顿。

### 标记清除算法

这种算法将判断”该对象是否不再需要“简化为”该对象是否不再能够被访问（_an object is unreachable_）“。

这种算法假设有一组对象称为”根（_Roots_）“。JavaScript 中，根就是全局对象。垃圾回收器将定期从根对象开始，寻找所有被根对象引用的对象，然后被这些对象引用的类型，以此类推。在这个过程中，垃圾回收器会找到所有可被访问（_reachable_）的对象，且回收所有不再能够被访问的对象。

这个算法是对前一个的改进，因为当一个对象有 0 个引用时，可以说该对象也是不再能被访问的。但是反之不成立，例如上面提到过的循环引用。

截止 2012 年，所有现代浏览器都使用了基于标记清除算法的垃圾回收器。在过去几年里，针对 JavaScript 垃圾回收领域的改进都是基于该算法实现上的改进，而没有改变该算法本身，也没有改变”该对象是否不再需要“的简化定义。

#### 循环引用不再是一个问题

在上述的第一个例子中，在函数调用返回后，从全局对象来看，变量`x`和`y`指向的对象无法被任何对象访问。因此，它们会被垃圾回收器找到，占用的内存会被回收。

#### 限制：手动释放内存

有时候，手动决定何时释放以及释放多少内存会更方便。为了释放一个对象占用的内存，这个对象需要被显示地标记为”不可访问“的。

截止到 2019 年，在 JavaScript 中，暂无方法显式地或以编程的方式触发垃圾回收。

## Node.js

Node.js 环境中提供了额外的配置项以及工具，可以用来排除一些内存问题。而在浏览器环境中执行的 JavaScript 可能无法使用这些配置项和工具。

### V8 引擎相关的命令行配置

可以通过如下命令来增加最大可供使用的堆内存：

`node --max-old-space-size=6000 index.js`

我们也可以结合 [Chrome Debugger](https://nodejs.org/en/docs/guides/debugging-getting-started/)，并使用如下命令来暴露垃圾回收器，从而排查一些内存有关的问题：

```sh
node --expose-gc --inspect index.js
```

## 参考

- [IBM article on "Memory leak patterns in JavaScript" (2007)](http://www.ibm.com/developerworks/web/library/wa-memleak/)
- [Kangax article on how to register event handler and avoid memory leaks (2010)](https://msdn.microsoft.com/en-us/magazine/ff728624.aspx)
- [Performance](https://developer.mozilla.org/en-US/docs/Mozilla/Performance)
