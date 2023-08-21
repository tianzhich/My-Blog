---
title: "【译】继承与原型链（Inheritance and the prototype chain）"
date: 2020-12-06T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["JavaScript", "MDN", "翻译", "原型链", "继承", "原型"]
categories: ["编程"]
series: ["mdn-advance-topics"]
featuredImage: "featured-image.webp"
---

继承与原型链（Inheritance and the prototype chain）。

<!--more-->

{{< admonition abstract "写在前面" >}}
原文来自 [MDN JavaScript 主题的高阶教程](https://developer.mozilla.org/en-US/docs/Web/JavaScript#advanced)部分，一共 5 篇，分别涉及继承和原型链、严格模式、类型数组、内存管理、并发模型和事件循环。本篇是第 1 篇，关于[继承与原型链](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)。

**2023-06-23 更新：** 今天发现，MDN 原文结构进行了调整，原来[高级教程](https://developer.mozilla.org/en-US/docs/Web/JavaScript#advanced)的 5 篇文章只剩下了 3 篇，分别是继承和原型链、内存管理、并发模型和事件循环，其中，严格模式被转移到了 [References/Misc](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference#additional_reference_pages) 下，类型数组被转移到了 [JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide#typed_arrays) 下。
{{< /admonition >}}

对于熟悉基于类的编程语言（例如 Java 和 C++）的开发者来说，JavaScript 会让他们感到困惑，因为 JS 的动态性以及其本身并不提供`class`的实现（ES2015 中提出的`class`关键字仅仅是语法糖，JS 仍然是基于原型的）

提到继承，JavaScript 只有一个结构：对象（objects）。每个对象都有一个私有属性，该属性链接到另一个对象（称为该对象的原型（**prototype**））。这个原型对象自身也有一个原型，直到一个对象的原型为`null`。根据定义，`null`不存在原型，它代表这条原型链的终点。

在 JavaScript 中，几乎所有对象都是[`Object`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)的实例，`Object`在原型链顶端。

尽管这种困惑经常被认为是 JavaScript 的缺点，但是这种原型式的继承模型实际上比一些经典的模型更为强大。例如，在一个原型式模型的基础上再构造一个经典模型是非常简单的。

## 通过原型链继承

### 继承属性

JavaScript 对象就像一堆属性的动态“包裹”（这堆属性称为对象自身属性）（译者注：原文为 _JavaScript objects are dynamic "bags" of properties (referred to as own properties)._）。
JavaScript 对象有一个指向原型对象的链接。当访问一个对象的属性时，不仅会在该对象上查找，还会在该对象的原型，以及这个原型的原型上查找，直到匹配上这个属性名或者遍历完该原型链。

> 根据 ECMAScript 标准，`someObject.[[Prototype]]`用于指定`someObject`的原型。从 ECMAScript 2015 开始，`[[Prototype]]`可以通过[`Object.getPrototypeOf()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getPrototypeOf)和[`Object.setPrototypeOf()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf)访问。这和通过 JavaScript 中的`__proto__`访问是一样的，尽管这不标准，但是已经被很多浏览器所实现。
> 最好不要和函数的<code>_func_.prototype</code>属性混淆。当一个函数被当做构造器（constructor）调用时，会生成一个对象，而函数上的<code>_func_.prototype</code>属性引用的对象会作为生成对象的`[[Prototype]]`存在。**`Object.prototype`**就表示了`Object`这一函数的 prototype。

下面例子展示了访问对象属性的过程：

```js
// 让我们使用构造函数f创建一个对象o，o上面有属性a和b:
let f = function () {
  this.a = 1;
  this.b = 2;
};
let o = new f(); // {a: 1, b: 2}

// 在f的prototype对象上添加一些属性
f.prototype.b = 3;
f.prototype.c = 4;

// 不要对prototype重新赋值比如： f.prototype = {b:3,c:4}; 这会打断原型链
// o.[[Prototype]] 上有属性b和c
// o.[[Prototype]].[[Prototype]] 就是 Object.prototype
// 最终, o.[[Prototype]].[[Prototype]].[[Prototype]] 为 null
// 这就是原型链的终端, 等于 null,
// 根据定义, null不再有 [[Prototype]]
// 因此, 整条原型链看起来类似:
// {a: 1, b: 2} ---> {b: 3, c: 4} ---> Object.prototype ---> null

console.log(o.a); // 1
// o上存在自身属性'a'吗？当然，该属性值为1

console.log(o.b); // 2
// o上存在自身属性'b'吗？当然，该属性值为2
// prototype 上也有属性'b', 但是并不会被访问到
// 这叫做“属性覆盖”

console.log(o.c); // 4
// o上存在自身属性'c'吗？不存在, 继续查找它的原型
// o.[[Prototype]]上存在自身属性'c'吗？当然，该属性值为4

console.log(o.d); // undefined
// o上存在自身属性'd'吗？不存在, 继续查找它的原型
// o.[[Prototype]]上存在自身属性'd'吗？不存在, 继续查找o.[[Prototype]]的原型
// o.[[Prototype]].[[Prototype]] 为 Object.prototype, 上面不存在属性'd', 继续查找o.[[Prototype]].[[Prototype]]的原型
// o.[[Prototype]].[[Prototype]].[[Prototype]] 为 null, 停止查找
// 没找到属性'd'，返回undefined
```

[在线代码链接](https://repl.it/@khaled_hossain_code/prototype)

在一个对象上设置属性称为创建了一个”自身属性“（译者注：原文为*Setting a property to an object creates an own property.*）。唯一会影响属性 set 和 get 行为的是当该属性使用[getter 或者 setter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Working_with_Objects#Defining_getters_and_setters)定义。

### 继承“方法”

JavaScript 中并没有像在基于类语言中定义的”方法“。在 JavaScript 中，任何函数也是以属性的形式被添加到对象中，继承的函数和其他继承的属性一样，也存在上面提到的”属性覆盖”（这里叫做*方法覆盖*（_method overriding_））。

当一个继承的函数被执行时，函数内的[`this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)指向当前继承的对象，而不一定是将该函数作为“自身属性“的对象本身。

```js
var o = {
  a: 2,
  m: function () {
    return this.a + 1;
  },
};

console.log(o.m()); // 3
// 当调用 o.m 时, 'this' 指向 o

var p = Object.create(o);
// p 是一个继承o的对象

p.a = 4; // 在p上创建一个'a'属性
console.log(p.m()); // 5
// 当调用 p.m 时, 'this' 指向 p.
// 所以当 p 从 o 上继承了方法 m时,
// 'this.a' 等于 p.a
```

---

## 在 JavaScript 中使用原型

让我们更详细地来看看背后的原理。

在 JavaScript 中，正如上面提到，函数也可以拥有属性。所有函数都有一个特殊的属性`prototype`。请注意下面的代码是独立的（可以安全地假设网页中除了下面的代码就没有其他代码了）。为了更好的学习体验，非常推荐你打开浏览器的控制台，点击'console'标签，复制粘贴以下代码，点击 Enter/Return 键来执行它。（大多数浏览器的开发者工具（Developer Tools）中都包含控制台。详情请查看[Firefox 开发者工具](https://wiki.developer.mozilla.org/en-US/docs/Tools)、[Chrome 开发者工具](https://developers.google.com/web/tools/chrome-devtools/)，以及[Edge 开发者工具](Edge DevTools)）

```js
function doSomething() {}
console.log(doSomething.prototype);
//  不管你如何声明函数,
//  JavaScript中的函数都有一个默认的
//  prototype 属性
//  (Ps: 这里有一个意外，箭头函数上没有默认的 prototype 属性)
var doSomething = function () {};
console.log(doSomething.prototype);
```

可以在 console 中看到，`doSomething()`有一个默认的`prototype`属性，打印的内容和下面类似：

```js
{
    constructor: ƒ doSomething(),
    __proto__: {
        constructor: ƒ Object(),
        hasOwnProperty: ƒ hasOwnProperty(),
        isPrototypeOf: ƒ isPrototypeOf(),
        propertyIsEnumerable: ƒ propertyIsEnumerable(),
        toLocaleString: ƒ toLocaleString(),
        toString: ƒ toString(),
        valueOf: ƒ valueOf()
    }
}
```

如果我们在`doSomething()`的`prototype`上添加属性，如下：

```js
function doSomething() {}
doSomething.prototype.foo = "bar";
console.log(doSomething.prototype);
```

结果为：

```js
{
    foo: "bar",
    constructor: ƒ doSomething(),
    __proto__: {
        constructor: ƒ Object(),
        hasOwnProperty: ƒ hasOwnProperty(),
        isPrototypeOf: ƒ isPrototypeOf(),
        propertyIsEnumerable: ƒ propertyIsEnumerable(),
        toLocaleString: ƒ toLocaleString(),
        toString: ƒ toString(),
        valueOf: ƒ valueOf()
    }
}
```

现在我们可以通过`new`操作符来基于这个 prototype 对象创建`doSomething()`的实例。使用`new`操作符调用函数只需要在调用前加上`new`前缀。这样该函数会返回其自身的一个实例对象。接着我们便可以往该实例对象上添加属性：

```js
function doSomething() {}
doSomething.prototype.foo = "bar"; // 往prototype上添加属性'foo'
var doSomeInstancing = new doSomething();
doSomeInstancing.prop = "some value"; // 往实例对象上添加属性'prop'
console.log(doSomeInstancing);
```

打印结果和如下类似：

```js
{
    prop: "some value",
    __proto__: {
        foo: "bar",
        constructor: ƒ doSomething(),
        __proto__: {
            constructor: ƒ Object(),
            hasOwnProperty: ƒ hasOwnProperty(),
            isPrototypeOf: ƒ isPrototypeOf(),
            propertyIsEnumerable: ƒ propertyIsEnumerable(),
            toLocaleString: ƒ toLocaleString(),
            toString: ƒ toString(),
            valueOf: ƒ valueOf()
        }
    }
}
```

可以得知，`doSomeInstancing`的`__proto__`就是`doSomething.prototype`。但是，这代表什么呢？放你访问`doSomeInstancing`的一个属性时，浏览器会首先查看`doSomeInstancing`自身是否存在该属性。

如果不存在，浏览器会继续查找`doSomeInstancing`的`__proto__`（或者说是 `doSomething.prototype`）。如果存在，则`doSomeInstancing`的`__proto__`的这个属性会被使用。

否则，会继续查找`doSomeInstancing`的`__proto__`的`__proto__`。默认情况下，任何函数 prototype 属性的`__proto__`属性就是`window.Object.prototype`。因此，会在`doSomeInstancing`的`__proto__`的`__proto__`（或者说是`doSomething.prototype.__proto__`，或者说是`Object.prototype`）继续查找对应属性。

最终，直到所有的`__proto__`被查找完毕，浏览器会断言该属性不存在，因此得出结论：该属性的值为 undefined。

然我们在 console 上再添加一些代码：

```js
function doSomething() {}
doSomething.prototype.foo = "bar";
var doSomeInstancing = new doSomething();
doSomeInstancing.prop = "some value";
console.log("doSomeInstancing.prop:      " + doSomeInstancing.prop);
console.log("doSomeInstancing.foo:       " + doSomeInstancing.foo);
console.log("doSomething.prop:           " + doSomething.prop);
console.log("doSomething.foo:            " + doSomething.foo);
console.log("doSomething.prototype.prop: " + doSomething.prototype.prop);
console.log("doSomething.prototype.foo:  " + doSomething.prototype.foo);
```

结果如下：

```text
doSomeInstancing.prop:      some value
doSomeInstancing.foo:       bar
doSomething.prop:           undefined
doSomething.foo:            undefined
doSomething.prototype.prop: undefined
doSomething.prototype.foo:  bar
```

---

## 使用不同的方法创建对象和原型链

### 使用语法结构（字面量）创建对象

```js
var o = { a: 1 };

// 新创建的对象以 Object.prototype 作为它的 [[Prototype]]
// o 没有叫做'hasOwnProperty'的自身属性
// hasOwnProperty 是 Object.prototype 的自身属性
// 也就是说 o 从Object.prototype 上继承了 hasOwnProperty
// Object.prototype 的原型为 null
// o ---> Object.prototype ---> null

var b = ["yo", "whadup", "?"];

// 数组继承自 Array.prototype
// (Array.prototype 上拥有方法例如 indexOf, forEach 等等)
// 原型链如下:
// b ---> Array.prototype ---> Object.prototype ---> null

function f() {
  return 2;
}

// 函数继承自 Function.prototype
// (Function.prototype 上拥有方法例如 call, bind, 等等)
// f ---> Function.prototype ---> Object.prototype ---> null
```

### 使用构造器函数

构造器函数和普通函数的差别就在于其恰好使用[`new`操作符](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new)调用

```js
function Graph() {
  this.vertices = [];
  this.edges = [];
}

Graph.prototype = {
  addVertex: function (v) {
    this.vertices.push(v);
  },
};

var g = new Graph();
// g 是一个有 'vertices' 和 'edges' 作为属性的对象
// 当执行 new Graph() 时，g.[[Prototype]] 的值就是 Graph.prototype
```

### 使用 `Object.create`

ECMAScript 提出了一个新方法：[`Object.create()`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)。调用该方法时会创建一个新对象。这个对象的原型为传入该函数的第一个参数：

```js
var a = { a: 1 };
// a ---> Object.prototype ---> null

var b = Object.create(a);
// b ---> a ---> Object.prototype ---> null
console.log(b.a); // 1 (继承自 a )

var c = Object.create(b);
// c ---> b ---> a ---> Object.prototype ---> null

var d = Object.create(null);
// d ---> null
console.log(d.hasOwnProperty);
// undefined, 因为 d 并没有继承自 Object.prototype
```

### 与`Object.create`和`new`操作符一起，使用`delete`操作符

下面的示例使用`Object.create`创建一个对象，并使用`delete`操作符来展示原型链的变化

```js
var a = { a: 1 };

var b = Object.create(a);

console.log(a.a); // 1
console.log(b.a); // 1
b.a = 5;
console.log(a.a); // 1
console.log(b.a); // 5
delete b.a;
console.log(a.a); // 1
console.log(b.a); // 1(b.a 的值 5 已经被删除，因此展示其原型链上的值)
delete a.a; // 也可以使用 'delete b.__proto__.a'
console.log(a.a); // undefined
console.log(b.a); // undefined
```

如果换成`new`操作符创建对象，原型链更短：

```js
function Graph() {
  this.vertices = [4, 4];
}

var g = new Graph();
console.log(g.vertices); // print [4,4]
g.vertices = 25;
console.log(g.vertices); // print 25
delete g.vertices;
console.log(g.vertices); // print undefined
```

### 使用 class 关键字

ECMAScript 2015 提出了一系列新的关键字用于实现[类](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)。包括[`class`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/class)、[`constructor`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/constructor)、[`static`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/static)、[`extends`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/extends)以及[`super`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super)。

```js
"use strict";

class Polygon {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
}

class Square extends Polygon {
  constructor(sideLength) {
    super(sideLength, sideLength);
  }
  get area() {
    return this.height * this.width;
  }
  set sideLength(newLength) {
    this.height = newLength;
    this.width = newLength;
  }
}

var square = new Square(2);
```

### 关于性能

如果需要查找的对象属性位于原型链的顶端，查找时间会对性能有影响，尤其对于对性能要求很高的应用来说，影响会进一步放大。另外，如果是访问一个不存在的属性，总是会遍历整条原型链。

此外，当对对象的属性进行迭代查找时，原型链上所有可枚举的属性都会被遍历。为了检查哪些属性是对象的自身属性而不是来自其原型链，很有必要使用继承自`Object.prototype`的[`hasOwnProperty`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwnProperty)方法。下面来看一个具体的例子，该例子继续使用上一个图形的例子：

```js
console.log(g.hasOwnProperty("vertices"));
// true

console.log(g.hasOwnProperty("nope"));
// false

console.log(g.hasOwnProperty("addVertex"));
// false

console.log(g.__proto__.hasOwnProperty("addVertex"));
// true
```

[`hasOwnProperty`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwnProperty)是 JavaScript 中查找对象属性时唯一不遍历原型链的方法。

注意：仅仅检查属性是[`undefined`](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/undefined)并不能代表该属性不存在，也许是因为它的值恰好被设置为了`undefined`。

### 不好的实践：对原生的 prototypes 进行扩展

经常容易犯的一个错误是扩展`Object.prototype`或者是一些其他内置的 prototype。

这被称为是”猴子补丁“，会打破程序的封装性。尽管在一些出名的框架中也这样做，例如 Prototype.js，但是仍然没有理由在内置类型上添加非标准的功能。

扩展内置类型的唯一理由是保证一些早期 JavaScript 引擎的兼容性，例如`Array.forEach`（译者注：`Array.forEach`是在 ECMA-262-5 中提出，部分早期浏览器引擎没有实现该标准，因此需要 polyfill）

### 继承原型链的方法总结

下面表格展示了四种方法以及它们各自的优缺点。以下例子创建的`inst`对象完全一致（因此控制台打印的结果也一样），除了它们之间有不同的优缺点。

<table>
    <tr>
        <th>名称</th>
        <th>举例</th>
        <th>优点</th>
        <th>缺点</th>
    </tr>
    <tr>
        <td>使用<code>new</code>初始化</td>
        <td>
            <pre lang="javascript">
                function foo(){}
                foo.prototype = {
                    foo_prop: "foo val"
                };
                function bar(){}
                var proto = new foo;
                proto.bar_prop = "bar val";
                bar.prototype = proto;
                var inst = new bar;
                console.log(inst.foo_prop);
                console.log(inst.bar_prop);
            </pre>
        </td>
        <td>支持所有浏览器（甚至到IE 5.5），同时，运行速度、标准化以及JIT优化性都非常好</td>
        <td>
            问题是，为了使用该方法函数必须被初始化。在初始化过程中，构造函数可能会为每个创建对象创建一些特有属性，然而例子中只会构造一次，因此这些特有信息只会生成一次，可能存导致潜在问题。
            之外，构造函数初始化时可能会添加冗余的方法到实例对象上。不过，只要这是你自己的代码且你明确这是干什么的，这些通常来说也不是问题（实际上是利大于弊）。
        </td>
    </tr>
    <tr>
        <td>使用<code>Object.create</code></td>
        <td>
            <pre lang="javascript">
                function foo(){}
                foo.prototype = {
                    foo_prop: "foo val"
                };
                function bar(){}
                var proto = Object.create(
                    foo.prototype
                );
                proto.bar_prop = "bar val";
                bar.prototype = proto;
                var inst = new bar;
                console.log(inst.foo_prop);
                console.log(inst.bar_prop);
            </pre>
            <pre lang="javascript">
                function foo(){}
                foo.prototype = {
                    foo_prop: "foo val"
                };
                function bar(){}
                var proto = Object.create(
                    foo.prototype,
                    {
                        bar_prop: {
                            value: "bar val"
                        }
                    }
                );
                bar.prototype = proto;
                var inst = new bar;
                console.log(inst.foo_prop);
                console.log(inst.bar_prop)
            </pre>
        </td>
        <td>支持目前所有的现代浏览器，包括非IE浏览器以及IE9及以上版本浏览器。相当于允许一次性设置<code>__proto__</code>，这样有利于浏览器优化该对象。同时也允许创建没有原型的对象例如：<code>Object.create(null)</code></td>
        <td>
            不支持IE8以及以下版本浏览器，不过，微软目前已不再支持运行这些浏览器的操作系统，对大多数应用来说这也不是一个问题。
            之外，如果使用第二个参数，则对象的初始化会变慢，这也许会成为性能瓶颈，因为第二个参数作为对象描述符属性，每个对象的描述符属性是另一个对象。当以对象形式处理成千上万的对象描述符时，可能会严重影响运行速度。
        </td>
    </tr>
    <tr>
        <td>使用<code>Object.setPrototypeOf</code></td>
        <td>
            <pre lang="javascript">
                function foo(){}
                foo.prototype = {
                    foo_prop: "foo val"
                };
                function bar(){}
                var proto = {
                    bar_prop: "bar val"
                };
                Object.setPrototypeOf(
                    proto, foo.prototype
                );
                bar.prototype = proto;
                var inst = new bar;
                console.log(inst.foo_prop);
                console.log(inst.bar_prop);
            </pre>
            <pre lang="javascript">
                function foo(){}
                foo.prototype = {
                    foo_prop: "foo val"
                };
                function bar(){}
                var proto;
                proto = Object.setPrototypeOf(
                    { bar_prop: "bar val" },
                    foo.prototype
                );
                bar.prototype = proto;
                var inst = new bar;
                console.log(inst.foo_prop);
                console.log(inst.bar_prop)
            </pre>
        </td>
        <td>支持目前所有的现代浏览器，包括非IE浏览器以及IE9及以上版本浏览器。支持动态的操作对象的原型，甚至可以为<code>Object.create(null)</code>创建的对象强制添加一个原型</td>
        <td>
            由于性能不佳，应该会被弃用。如果你敢在生产环境中使用这样的语法，JavaScript代码快速运行几乎不可能。因为许多浏览器优化了原型，举个例子，在访问一个对象上的属性之前，编译器会提前确定原型上的属性在内存中的位置，但是如果使用了<code>Object.setPrototypeOf</code>对原型进行动态更改，这相当于扰乱了优化，甚至会让编译器重新编译并放弃对这部分的优化，仅仅是为了能让你这段代码跑起来。
            同时，不支持IE8以及以下版本浏览器
        </td>
    </tr>
    <tr>
        <td>使用<code>__proto__</code></td>
        <td>
            <pre lang="javascript">
                function foo(){}
                foo.prototype = {
                    foo_prop: "foo val"
                };
                function bar(){}
                var proto = {
                    bar_prop: "bar val",
                    __proto__: foo.prototype
                };
                bar.prototype = proto;
                var inst = new bar;
                console.log(inst.foo_prop);
                console.log(inst.bar_prop);
            </pre>
            <pre lang="javascript">
                var inst = {
                    __proto__: {
                        bar_prop: "bar val",
                        __proto__: {
                            foo_prop: "foo val",
                            __proto__: Object.prototype
                        }
                    }
                };
                console.log(inst.foo_prop);
                console.log(inst.bar_prop)
            </pre>
        </td>
        <td>支持目前几乎所有的现代浏览器，包括非IE浏览器以及IE11及以上版本浏览器。将<code>__proto__</code>设置为非对象的类型不会抛出异常，但是会导致程序运行失败</td>
        <td>
            严重过时而且性能不佳。如果你敢在生产环境中使用这样的语法，JavaScript代码快速运行几乎不可能。因为许多浏览器优化了原型，举个例子，在访问一个对象上的属性之前，编译器会提前确定原型上的属性在内存中的位置，但是如果使用了<code>__proto__</code>对原型进行动态更改，这相当于扰乱了优化，甚至会让编译器重新编译并放弃对这部分的优化，仅仅是为了能让你这段代码跑起来。
            同时，不支持IE10及以下版本浏览器。
        </td>
    </tr>
</table>

---

## `prototype`和`Object.getPrototypeOf`

对于从 Java 和 C++过来的开发者来说，JavaScript 会让他们感到有些困惑，因为 JavaScript 是动态类型、代码无需编译可以在 JS Engine 直接运行（译者注：Java 代码需要编译成机器码后在 JVM 执行），同时它还没有类。所有的几乎都是实例（objects）。尽管模拟了`class`，但其本质还是函数对象。

你也许注意到了`function A`上有一个特殊的属性`prototype`。这个特殊属性与 JavaScript`new`操作符一起使用。当使用`new`操作符创建出来一个实例对象，这个特殊属性`prototype`会被复制给该对象的内部`[[Prototype]]`属性。举个例子，当运行`var a1 = new A()`代码时，JavaScript（在内存中创建完新实例对象之后且准备运行函数`A()`之前，运行函数时函数内部的`this`会指向该对象）会设置：`a1.[[Prototype]] = A.prototype`。
当你之后访问创建的对象属性时，JavaScript 首先会检查属性是否存在于对象本身，如果不存在，则继续查找其`[[Prototype]]`。这意味着你在`prototype`上定义的属性实际上被所有实例对象共享，如果你愿意，甚至可以修改`prototype`，这些改动会同步到所有存在的实例对象中。

如果在上面的例子中，你执行：`var a1 = new A(); var a2 = new A();`，那么`a1.doSomething`就是`Object.getPrototypeOf(a1).doSomething`，这和你定义的`A.prototype.doSomething`是同一个对象，所以：`Object.getPrototypeOf(a1).doSomething === Object.getPrototypeOf(a2).doSomething === A.prototype.doSomething`。

简而言之，`prototype`是针对类型的，而`Object.getPrototypeOf()`对于实例对象是一致的。（译者注：原文为*In short, `prototype` is for types, while `Object.getPrototypeOf()` is the same for instances.*）。

`[[Prototype]]`会被递归地查找，例如：`a1.doSomething`, `Object.getPrototypeOf(a1).doSomething`, `Object.getPrototypeOf(Object.getPrototypeOf(a1)).doSomething`等等，直到`Object.getPrototypeOf`返回`null`。

因此，当你执行：

```js
var o = new Foo();
```

实际上是执行：

```js
var o = new Object();
o[[Prototype]] = Foo.prototype;
Foo.call(o);
```

接着如果你访问：

```js
o.someProp;
```

JavaScript 会检查是否 o 上存在自身属性`someProp`。如果不存在，继续检查`Object.getPrototypeOf(o).someProp`是否存在，如果还不存在继续检查`Object.getPrototypeOf(Object.getPrototypeOf(o)).someProp`，依次类推。

---

## 总结

在编写基于原型的复杂代码之前，很有必要先理解原型式的继承模型。同时，请注意代码中原型链的长度，并且在必要时将其分解以避免可能存在的性能问题。此外，应该杜绝在原生的原型对象上进行扩展，除非是为了考虑兼容性，例如在老的 JavaScript 引擎上适配新的语言特性。

---

**Tags:** [Advanced](https://wiki.developer.mozilla.org/en-US/docs/tag/Advanced) [Guide](https://wiki.developer.mozilla.org/en-US/docs/tag/Guide) [Inheritance](https://wiki.developer.mozilla.org/en-US/docs/tag/Inheritance) [JavaScript](https://wiki.developer.mozilla.org/en-US/docs/tag/JavaScript) [OOP](https://wiki.developer.mozilla.org/en-US/docs/tag/OOP)
