---
title: "【译】严格模式（Strict mode）"
date: 2020-12-27T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["JavaScript", "MDN", "翻译", "严格模式"]
categories: ["编程"]
series: ["translations", "mdn-advance-topics"]
featuredImage: "featured-image.webp"
---

严格模式（Strict mode）。

<!--more-->

{{< admonition abstract "写在前面" >}}
原文来自 [MDN JavaScript 主题的高阶教程](https://developer.mozilla.org/en-US/docs/Web/JavaScript#advanced)部分，一共 5 篇，分别涉及继承和原型链、严格模式、类型数组、内存管理、并发模型和事件循环。本篇是第 2 篇，关于[严格模式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)。

**2023-06-23 更新：** 今天发现，MDN 原文结构进行了调整，原来[高级教程](https://developer.mozilla.org/en-US/docs/Web/JavaScript#advanced)的 5 篇文章只剩下了 3 篇，分别是继承和原型链、内存管理、并发模型和事件循环，其中，严格模式被转移到了 [References/Misc](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference#additional_reference_pages) 下，类型数组被转移到了 [JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide#typed_arrays) 下。
{{< /admonition >}}

ECMAScript 5 中推出的 JavaScript 严格模式（strict mode），可以让你使用 JavaScript 的一种受限”变体“，从而悄悄地退出了”正常模式（[sloppy mode](https://developer.mozilla.org/docs/Glossary/Sloppy_mode)）“。严格模式并非只是正常模式的子集：它专门拥有与正常模式下的代码不一样的语义。如果浏览器 A 不支持严格模式，浏览器 B 支持，它们俩运行同样的严格模式代码结果也会不同，所以如果没有对严格模式下的代码进行相应的功能测试，请不要依赖它。严格模式和正常模式的代码可以共存，所以正常模式的代码可以逐渐改造，最终替换为严格模式代码。

> 有时候你会看到正常的，非严格的模式也被称作”**马虎模式（[sloppy mode](https://developer.mozilla.org/docs/Glossary/Sloppy_mode)）**“，这并不是术语，但知道总归是好的。

相比正常模式，严格模式在 JavaScript 语义上做了如下改动：

1. 消除了一些 JavaScript 隐蔽的错误，并将它们抛出来。
2. 修复了一些 JavaScript 错误（这些错误使得 JavaScript 引擎难以优化代码）：有时候严格模式下的代码比正常模式执行得更快。
3. 对于在未来版本的 ECMAScript 中可能出现的功能，在语法上进行禁止（例如禁止使用某些关键字）。

如果你想更改代码以让其在 JavaScript 的这个受限”变体“中运行，可以查看”[迁移到严格模式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode/Transitioning_to_strict_mode)“

---

## 启用严格模式

严格模式可以在*全局作用域*或*函数作用域*启用，但是无法在块级作用域 `{}` 中单独启用，如果尝试在其中启用不会有任何效果。`eval`代码、`Function`代码、事件处理程序属性（译者注：例如`<button onclick="document.getElementById('demo').innerHTML = Date()">The time is?</button>`）、[`WindowTimers.setTimeout()`](https://developer.mozilla.org/en-US/docs/Web/API/WindowTimers/setTimeout)的字符串参数（译者注：例如`var timeoutID = scope.setTimeout(code[, delay]);`）等等类似的代码均为*全局作用域*，在它们中也可以启用严格模式。

### *全局作用域*下的严格模式

想要在全局作用域下启用严格模式，只需要在任意其他代码之前写上`"use strict";`（或者`'use strict';`）。

```js
// 全局作用域下的严格模式语法
"use strict";
var v = "Hi! I'm a strict mode script!";
```

这个语法存在一个[陷阱](https://bugzilla.mozilla.org/show_bug.cgi?id=579119)，某个主要[网站](https://bugzilla.mozilla.org/show_bug.cgi?id=627531)受此影响：对于冲突的两个 script 无法进行合并。例如：如果需要将一个正常模式的 script 合并到一个严格模式的 script，那么整个 script 都会变成严格模式！反过来则会成为正常模式的 script。显然，合并脚本从来不是明智之举，如果非要这样做，请在每个函数作用域下单独启用。

你也可以将脚本的所有内容放在一个函数中，并在该函数中决定是否启用严格模式。这消除了合并的问题，也意味着你必须从函数中人工导出需要共享的变量。

### *函数作用域*下的严格模式

同样的，为了在函数作用域下启用严格模式，在函数体中任意代码前写上`"use strict";`（或者`'use strict';`）。

```js
function strict() {
  // 函数作用域下的严格模式语法
  "use strict";
  function nested() {
    return "And so am I!";
  }
  return "Hi!  I'm a strict mode function!  " + nested();
}
function notStrict() {
  return "I'm not strict.";
}
```

### 模块（modules）下的严格模式

ECMAScript 2015 推出了 JavaScript 模块（[JavaScript modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export)），所以现在有第三种方法启用严格模式。JavaScript 模块中的代码自动开启了严格模式，无需进行任何声明。

```js
function strict() {
  // 由于该函数位于模块内，代码自动启用严格模式
}
export default strict;
```

---

## 严格模式下的一些改变

严格模式既改变了句法（syntax），也改变了运行时表现（runtime behavior）。改变可以分为以下几类：

1. 将一些误解（mistake）转变为错误（error）（句法错误（syntax error）或运行时错误（runtime error））
2. 简化了变量名到特定变量的映射关系
3. 简化了`eval`和`arguments`的使用
4. 更容易书写”安全“的 JavaScript（secure JavaScript）
5. 对未来的 ECMAScript 的进化做铺垫（译者注：可以理解为限制一些未来规范中可能定义的一些关键字的使用）

### 将误解转变为错误

严格模式下，之前被广泛接受的误解被转换成错误。JavaScript 是一个对新手友好的开发语言，有时候一些本该是错误操作却被赋予了非错误的语义。这样做能够临时解决一些问题，但是有时也为将来埋下了更糟糕的问题。严格模式可以让这些错误的操作变成运行时错误，从而及时得到发现和解决。

第一，严格模式禁止了意外创建全局变量，在常规 JavaScript 中，如果赋值时误输入了一个变量，它会成为全局对象的属性并且程序继续”正常工作“（不过在 JavaScript 的未来版本中可能会运行失败）。严格模式下，如果为一个未定义的变量进行赋值操作，不会创建一个全局变量，而是抛出错误：

```js
"use strict";
// 假设全局变量 mistypeVariable 不存在
mistypeVariable = 17; // 由于误拼写了变量名，这一行会抛出引用错误（ReferenceError）
```

第二，严格模式下，之前一些赋值操作只是无效，现在则会抛出异常。例如，`NaN`是一个不可写的全局变量。常规模式下赋值给`NaN`不会发生任何事，开发者也不会收到任何错误的反馈。严格模式下这样做则会抛出异常。在严格模式下，任何在常规模式下无效的赋值操作（例如赋值给一个不可写的全局变量或对象属性、赋值给只读的对象属性、在[不可扩展的对象](https://developer.mozilla.org/en-US/Web/JavaScript/Reference/Global_Objects/Object/preventExtensions)上添加一个新属性）在严格模式下都将会抛出异常：

```js
"use strict";

// 赋值给一个不可写的全局变量
var undefined = 5; // 抛出 TypeError
var Infinity = 5; // 抛出 TypeError

// 赋值给一个不可写的对象属性
var obj1 = {};
Object.defineProperty(obj1, "x", { value: 42, writable: false });
obj1.x = 9; // 抛出 TypeError

// 赋值给一个只读的对象属性
var obj2 = {
  get x() {
    return 17;
  },
};
obj2.x = 5; // 抛出 TypeError

// 给不可扩展的对象添加一个新属性
var fixed = {};
Object.preventExtensions(fixed);
fixed.newProp = "ohai"; // 抛出 TypeError
```

第三，严格模式下，如果试图删除一个不可删除的属性也会抛出异常（常规模式下只是无效）：

```js
"use strict";
delete Object.prototype; // 抛出 TypeError
```

第四，在 Gecko 34 版本前的严格模式中，要求所有对象字面量上定义的属性唯一。而常规模式下只是复制一份属性，最后一个属性决定最终的取值。由于同名属性只有最后一个起作用，如果更改属性值时改的不是最后一个属性本身，这样的复制可能会导致一连串 bug。在严格模式下重复定义属性是语法错误：

> 注：ECMAScript 2015 后允许重复定义属性（[bug 1041128](https://bugzilla.mozilla.org/show_bug.cgi?id=1041128)）

```js
"use strict";
var o = { p: 1, p: 2 }; // !!! syntax error
```

第五，严格模式要求函数参数名唯一。而常规模式下最后一个参数会覆盖之前同名的，不过他们并非完全不可访问，通过`arguments[i]`仍然可以获取这些参数值。尽管如此，这种覆盖没有意义，甚至有可能是无意为之（例如可能只是拼写错误），所以严格模式下这样做是语法错误：

```js
function sum(a, a, c) {
  // !!! syntax error
  "use strict";
  return a + a + c; // 代码运行结果是错误的
}
```

第六，严格模式禁止八进制的语法。八进制语法并非 ECMAScript 标准，但是所有浏览器都支持该语法（只需在八进制数前面加上 0：`0644 === 420`，或者`"\045" === "%"`）。在 ECMAScript 2015 中可以通过加上`0o`成为八进制数，例如：

```js
var a = 0o10; // ES2015: 八进制
```

新手开发者可能认为 0 作为前缀并没有什么语义，所以他们使用 0 来进行代码对齐，但是这样改变了数字真正的值！以 0 作为前缀几乎没用，反而有误导作用，因此严格模式下这样做会导致语法错误：

```js
"use strict";
var sum =
  015 + // !!! syntax error
  197 +
  142;

var sumWithOctal = 0o10 + 8;
console.log(sumWithOctal); // 16
```

第七，严格模式禁止在[基础类型](https://developer.mozilla.org/en-US/docs/Glossary/primitive)值上添加属性，而常规模式这样操作只是无效。严格模式会抛出[TypeError](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypeError)：

```js
(function () {
  "use strict";

  false.true = ""; // TypeError
  (14).sailing = "home"; // TypeError
  "with".you = "far away"; // TypeError
})();
```

### 简化变量的使用

严格模式简化了变量名和代码中具体的变量定义间的映射关系（译者注：就是给定一个变量名，可以更简单地找到其定义（也就是内存地址），从而获取值）。编译器的一些优化依赖于快速找到某个变量存储在哪，这对于全面优化 JavaScript 代码至关重要。有时候这种这种映射关系直到运行时才能建立。严格模式消除了影响映射关系建立的大多数情况，因此编译器能更好地优化严格模式代码。

第一，严格模式禁止使用`with`，`with`的问题在于其代码块中的任何变量名要么映射到传给它的对象属性上，要么是他的外部作用域（可能是全局作用域）变量上，而且映射关系只能在运行时建立，无法提前知道。严格模式使`with`成为语法错误，从而避免其代码块中的变量名在代码运行前无法确定具体位置的问题：

```js
"use strict";
var x = 17;
with (obj) {
  // !!! syntax error
  // 如果不是在严格模式下，x 指的是外部定义的 `var x` ?
  // 又或者是 `obj.x` ? 在代码真正运行前无法准确得知，
  // 所以这段代码无法优化
  x;
}
```

一个简单的替代方案是将对象临时存储到一个变量，然后访问这个变量的对应属性以替代使用`with`。

第二，[严格模式下的`eval`中声明的变量不会泄露到其周围作用域](http://whereswalden.com/2011/01/10/new-es5-strict-mode-support-new-vars-created-by-strict-mode-eval-code-are-local-to-that-code-only/)。常规模式中`eval("var x;")`会在其周围作用域（函数或者全局作用域）中引入变量`x`。这意味着，一般情况下，在含有`eval`调用的函数中，非参数或局部变量的变量名也必须等到运行时能映射到具体变量（因为`eval`调用可能会引入新变量从而覆盖外部变量）。严格模式下`eval`仅为传入其中解析的代码创建变量，所以不会影响到外部或局部的变量：

**译者注：这段话的英文原文如下：**

> Second, [`eval of strict mode code does not introduce new variables into the surrounding scope`](http://whereswalden.com/2011/01/10/new-es5-strict-mode-support-new-vars-created-by-strict-mode-eval-code-are-local-to-that-code-only/). In normal code `eval("var x;")` introduces a variable `x` into the surrounding function or the global scope. This means that, in general, in a function containing a call to `eval` every name not referring to an argument or local variable must be mapped to a particular definition at runtime (because that `eval` might have introduced a new variable that would hide the outer variable). In strict mode `eval` creates variables only for the code being evaluated, so `eval` can't affect whether a name refers to an outer variable or some local variable:

> 读起来可能有点拗口，简而言之，就是在一个函数中调用了`eval`。函数中此时存在三种变量引用关系，第一种是局部定义变量，第二种是函数参数变量，第三种是外部变量（外部作用域可能仍然为函数，或者全局作用域）。对于前两种变量的取值映射肯定得等到函数执行时，但是对于第三种，正常在函数声明时就能确定取值，而由于引入`eval`存在变量泄露，此时如果泄露的变量是外部变量，则取值会发生改变。这样会导致第三种取值映射也得等到函数执行时才确定！因此上面的`local variable`指的就是前两种，`outer variable`指的第三种。

```js
var x = 17;
var evalX = eval("'use strict'; var x = 42; x;");
console.assert(x === 17);
console.assert(evalX === 42);
```

如果在一段严格模式代码中，`eval`函数以`eval(...)`这种表达式形式被调用，传入其中的字符串会被解析成严格模式代码并执行。也可以在传入的字符串上显式声明严格模式，但没必要：

```js
function strict1(str) {
  "use strict";
  return eval(str); // str 会被视为严格模式代码
}
function strict2(f, str) {
  "use strict";
  return f(str); // 不是 `eval(...)`这种表达式形式: 仅仅在 str 中显式声明严格模式下，
  // str 才会被视为严格模式代码
}
function nonstrict(str) {
  return eval(str); // 仅仅在 str 中显式声明严格模式下，
  // str 才会被视为严格模式代码
}

strict1("'Strict mode code!'");
strict1("'use strict'; 'Strict mode code!'");
strict2(eval, "'Non-strict code.'");
strict2(eval, "'use strict'; 'Strict mode code!'");
nonstrict("'Non-strict code.'");
nonstrict("'use strict'; 'Strict mode code!'");
```

因此，在 `eval` 中执行的严格模式代码下，变量的行为与严格模式下非 `eval` 执行的代码中的变量行为相同。

第三，严格模式禁止删除单一变量名，`delete name`在严格模式下是语法错误：

```js
"use strict";

var x;
delete x; // !!! syntax error

eval("var y; delete y;"); // !!! syntax error
```

### 使`eval`和`arguments`更简单

严格模式下`eval`和`arguments`不会再那么奇妙，而在常规模式下他俩都包含了很多奇妙的行为：`eval`可以添加和移除绑定，修改绑定的值，`arguments`可以使用它的索引属性名作为函数参数的别名。严格模式在将`eval`和`arguments`视为关键字方面取得了很大的进步，尽管完整的修复要到未来的 ECMAScript 版本中才会到来。

第一，在语法层面，`eval`和`arguments`不能被绑定或者赋值。所有这些操作都将是语法错误：

```js
"use strict";
eval = 17;
arguments++;
++eval;
var obj = { set p(arguments) {} };
var eval;
try {
} catch (arguments) {}
function x(eval) {}
function arguments() {}
var y = function eval() {};
var f = new Function("arguments", "'use strict'; return 17;");
```

第二，`arguments`对象的索引属性不再作为函数参数的别名。在常规模式的函数中，如果函数的第一个参数为`arg`，更改`arg`同样会影响`arguments[0]`，反之亦然（除非没有参数或者`arguments[0]`被删除）。严格模式函数中的`arguments`对象存储这函数被调用时的原始参数。`arguments[i]`不再追踪相应命名参数对应值的变化，而该命名参数也不再追踪`arguments[i]`对应的值的变化。

```js
function f(a) {
  "use strict";
  a = 42;
  return [a, arguments[0]];
}
var pair = f(17);
console.assert(pair[0] === 42);
console.assert(pair[1] === 17);
```

第三，不再支持`arguments.callee`。常规函数中`arguments.callee`指代函数本身。这样使用没有什么意义：只要给函数本身命名并使用这个名字就可以了！此外，`arguments.callee`严重阻碍了编译器优化，例如内联函数（inlining functions），因为访问`arguments.callee`，则必须维持对该函数的引用，从而无法内联。严格模式中，函数内访问或设置`arguments.callee`都将导致报错，而且它也是一个不可删除的属性：

```js
"use strict";
var f = function () {
  return arguments.callee;
};
f(); // 抛出 TypeError
```

### ”安全的“JavaScript

严格模式使得编写”安全的“JavaScript 变得更容易。现在一些网站提供给用户编写 JavaScript 的方式，然后由网站*代表其他用户*运行。浏览器中的 JavaScript 可以反问用户的私人信息，因此这类 JavaScript 在执行前必须经过一些转换，以审查其是否访问了某些禁止功能。JavaScript 的灵活特性使得如果不进行很多运行时检测，做到上述审查实际上是不可能的。部分语言的功能非常普遍，以至于执行运行时检查会有很大性能消耗。部分严格模式扭转了这种局面，加上要求用户提交的 JavaScript 代码为严格模式，以便以某种确认的方式调用，大大地减少了运行时检查的必要。

第一，函数中`this`的值不再强制是对象（称为包装（"boxed"））。常规模式的函数中，`this`总是对象：要么是调用时绑定到`this`的对象，要么是包装对象（如果是 string、bool 和 number），要么是全局对象（如果传入`undefined`或`null`）（使用[`call`](https://developer.mozilla.org/en-US/Web/JavaScript/Reference/Global_Objects/Function/call)、[`apply`](https://developer.mozilla.org/en-US/Web/JavaScript/Reference/Global_Objects/Function/apply)或[`bind`](https://developer.mozilla.org/en-US/Web/JavaScript/Reference/Global_Objects/Function/bind)可以指定绑定到`this`的对象）。自动包装不仅有性能损耗，同时暴露浏览器的全局对象还有安全隐患（因为全局对象提供的一些功能是”安全的“JavaScript 环境必须加以约束限制的）。所以在严格模式下的函数中，`this`不会被自动包装成对象，而且如果没有指定，`this`则为`undefined`。

```js
"use strict";
function fun() {
  return this;
}
console.assert(fun() === undefined);
console.assert(fun.call(2) === 2);
console.assert(fun.apply(null) === null);
console.assert(fun.call(undefined) === undefined);
console.assert(fun.bind(true)() === true);
```

这意味着，在严格模式的函数中无法再使用`this`来访问`window`对象。

第二，在严格模式中，无法再通过 ECMAScript 中的常用扩展功能来追踪 JavaScript 代码堆栈。在常规模式中，假设`fun`函数在中途被调用，通过这些扩展功能，例如`fun.caller`可以获取最近调用`fun`函数的函数，`fun.arguments`则可以获取此次调用的函数参数。这些扩展功能都可能危害”安全的“JavaScript 代码，因为它们允许”安全的“代码访问”专有“函数以及它们的（可能没有受到保护的）参数。如果`fun`位于严格模式中，`fun.caller`和`fun.arguments`均为不可删除的属性，同时读取和设置它们都会抛出错误。

```js
function restricted() {
  "use strict";
  restricted.caller; // 抛出 TypeError
  restricted.arguments; // 抛出 TypeError
}
function privilegedInvoker() {
  return restricted();
}
privilegedInvoker();
```

第三，严格模式下的`arguments`不再提供对当前调用函数的局部变量访问。在一些老的 ECMAScript 版本实现中，`arguments.caller`对象上的属性可作为该函数调用中局部变量的别名。这是一个[安全漏洞](http://stuff.mit.edu/iap/2008/facebook/)，因为它打破了通过函数抽象来隐藏私有变量的能力，同时也阻碍了大部分的编译器优化。基于以上原因，现在的浏览器都没有实现它。至今，由于`arguments.caller`的这些历史功能，严格模式下其也是不可删除的属性，设置或读取都会抛出错误。

```js
"use strict";
function fun(a, b) {
  "use strict";
  var v = 12;
  return arguments.caller; // 抛出 TypeError
}
fun(1, 2); // 不会暴露 v (或者 a 和 b)
```

### 为将来的 ECMAScript 铺平道路

未来的 ECMAScript 可能会推出新语法，ECMAScript 5 的严格模式使用了一些限制，以使得过渡更加平滑。如果未来的一些变更基础已经在严格模式中被禁止，那这些变更在新版本中的推进就会更加容易。

第一，在严格模式中，少部分标识符成为了保留关键字。这些包括：`implements`、`interface`、`let`、`package`、`private`、`protected`、`public`、`static`、`yield`。在严格模式中，你不能使用这些来进行变量或参数命名。

```js
function package(protected) {
  // !!!
  "use strict";
  var implements; // !!!

  // !!!
  interface: while (true) {
    break interface; // !!!
  }

  function private() {} // !!!
}
function fun(static) {
  "use strict";
} // !!!
```

有两条针对 Mozilla 浏览器的警告：首先，如果你的 JavaScript 版本大于或等于 1.7（例如你的 chrome 代码或正确使用了`<script type="">`（译者注：原文为*for example in chrome code or when using the right `<script type="">`*，这里不太清楚*chrome code*的意思。。。）），同时使用了严格模式，因为`let`和`yield`最先推出，所以它们能够正常工作。但是在网页中的严格模式代码，例如使用`<script src="">`或`<script>...</script>`加载的，则无法使用`let/yield`标识符。其次，尽管 ES5 无条件地保留了`class`、`enum`、`extends`、`import`以及`super`，在 Firefox 5 以前，Mozilla 仅仅在严格模式中保留了它们。

第二，[严格模式禁止函数声明在全局或函数顶层之外](http://whereswalden.com/2011/01/24/new-es5-strict-mode-requirement-function-statements-not-at-top-level-of-a-program-or-function-are-prohibited/)。在常规模式下的浏览器中，函数声明允许在”任何地方“。*这并不是 ES5（甚至 ES3）规范！*而是一个语法扩展，而且在不同浏览器中语义不兼容。注意 ECMAScript 2015 允许了顶层之外进行函数声明。

```js
"use strict";
if (true) {
  function f() {} // !!! syntax error
  f();
}

for (var i = 0; i < 5; i++) {
  function f2() {} // !!! syntax error
  f2();
}

function baz() {
  // 合法
  function eit() {} // 也合法
}
```

这种禁止正确来说不是严格模式，因为允许函数声明在”任何地方“是 ES5 的一个语法扩展。但是 ECMAScript 委员会推荐了这个语法扩展，因此浏览器都实现了它。

## 浏览器中的严格模式

主流浏览器现在都实现了严格模式。然而，由于目前仍然有[许多版本的浏览器仅仅部分支持或完全不支持严格模式](http://caniuse.com/use-strict)（例如 IE10 以下浏览器）。_严格模式改变了代码的语义_。如果浏览器不支持严格模式，依赖这些改变会导致一些误解或错误。使用严格模式时务必格外小心，依赖严格模式之前，请确保进行了单元测试以验证相关部分严格模式的代码正常运行。最后，确保在*支持和不支持严格模式的浏览器中均测试了代码*。如果你仅仅在不支持严格模式的浏览器中测试代码，你可能会在支持严格模式的浏览器中遇到一些问题，反之亦然。

## 规范说明

> [ECMAScript (ECMA-262)](https://tc39.es/ecma262/#sec-strict-mode-code)
> [The definition of 'Strict Mode Code' in that specification.](https://tc39.es/ecma262/#sec-strict-mode-code)

## 参考文章

- [Where's Walden? » New ES5 strict mode support: now with poison pills!](http://whereswalden.com/2010/09/08/new-es5-strict-mode-support-now-with-poison-pills/)
- [Where's Walden? » New ES5 strict mode requirement: function statements not at top level of a program or function are prohibited](http://whereswalden.com/2011/01/24/new-es5-strict-mode-requirement-function-statements-not-at-top-level-of-a-program-or-function-are-prohibited/)
- [Where's Walden? » New ES5 strict mode support: new vars created by strict mode eval code are local to that code only](http://whereswalden.com/2011/01/10/new-es5-strict-mode-support-new-vars-created-by-strict-mode-eval-code-are-local-to-that-code-only/)
- [JavaScript "use strict" tutorial for beginners.](http://qnimate.com/javascript-strict-mode-in-nutshell/)
- [John Resig - ECMAScript 5 Strict Mode, JSON, and More](http://ejohn.org/blog/ecmascript-5-strict-mode-json-and-more/)
- [ECMA-262-5 in detail. Chapter 2. Strict Mode.](http://dmitrysoshnikov.com/ecmascript/es5-chapter-2-strict-mode/)
- [Strict mode compatibility table](http://kangax.github.io/compat-table/es5/#Strict_mode)
- [Transitioning to strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode/Transitioning_to_strict_mode)
