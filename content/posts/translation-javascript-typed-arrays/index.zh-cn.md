---
title: "【译】JavaScript 类型数组（JavaScript typed arrays）"
date: 2020-12-27T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["JavaScript", "MDN", "翻译", "类型数组", "数组"]
categories: ["编程"]
series: ["translations", "mdn-advance-topics"]
featuredImage: "featured-image.webp"
---

JavaScript 类型数组（JavaScript typed arrays）。

<!--more-->

{{< admonition abstract "写在前面" >}}
原文来自 [MDN JavaScript 主题的高阶教程](https://developer.mozilla.org/en-US/docs/Web/JavaScript#advanced)部分，一共 5 篇，分别涉及继承和原型链、严格模式、类型数组、内存管理、并发模型和事件循环。本篇是第 3 篇，关于[类型数组](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Typed_arrays)。

**2023-06-23 更新：** 今天发现，MDN 原文结构进行了调整，原来[高级教程](https://developer.mozilla.org/en-US/docs/Web/JavaScript#advanced)的 5 篇文章只剩下了 3 篇，分别是继承和原型链、内存管理、并发模型和事件循环，其中，严格模式被转移到了 [References/Misc](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference#additional_reference_pages) 下，类型数组被转移到了 [JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide#typed_arrays) 下。
{{< /admonition >}}

**JavaScript 类型数组**是一种类数组对象，它们提供了一种在内存缓冲区中读写二进制数据的机制。你也许已经知道，[`Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)大小能够动态伸缩而且可以存储 JavaScript 的任何类型。JavaScript 引擎进行了一些优化，使得操作这些数组很快。

然而，随着 web 应用变得越来越强大，如果需要一些额外功能，例如音视频数据操作、使用 WebSockets 访问原始数据等等，很明显，如果 JavaScript 能够快速轻松地操作原始二进制数据会很有帮助。这就是类型数据提出的原因。JavaScript 类型数组的每一项都是某种格式的二进制数据，这些数据格式范围从 8 位整数到 64 位浮点数。

然而，不要吧类型数组与普通数组混淆，因为使用[`Array.isArray()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray)检测类型数组会返回`false`。除此之外，并不是所有普通数组上的方法都存在于类型数组上（例如 push 和 pop）。

## 缓冲区（Buffers）和视图（Views）：类型数组结构

为了最大化可扩展性和提升效率，JavaScript 类型数组被分成两个部分：缓冲区（buffers）和视图（views）。缓冲区（由[`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)对象实现）是一个包含着一大块数据的对象，它没有任何格式，也没提供机制来访问其内容。为了访问缓冲区中的内存，你需要使用视图。视图提供了一个上下文，其中包含了一种数据类型、起始偏移量以及元素数量。视图将数据转化为一个类型数组。

![ArrayBuffer](./pic-1.png)

### ArrayBuffer

[ArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)是一种数据类型，其被用于表示一块通用且固定大小的二进制数据缓冲区。你无法直接操作`ArrayBuffer`的内容，而是需要创建一个类型数组视图或者是一个[`DataView`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView)，它们以一种特殊格式表示着这块缓冲区，同时可以使用它们来读写缓冲区中的内容。

### 类型数组视图

类型数组的视图具有一个描述性的名字，它们为所有通用的数值类型（例如`Int8`、`Uint32`、`Float64`等等）提供了一种视图。还有一种特殊的视图叫做`Uint8ClampedArray`，它将取值控制在 0-255 之间，这有助于[Canvas 的数据处理](https://developer.mozilla.org/en-US/docs/Web/API/ImageData)。

| 类型                                                                                                                    | 取值范围                                      | 大小（单位：字节） | 说明                                                                                                             | [**Web IDL** 类型](https://en.wikipedia.org/wiki/Web_IDL) | 对应的 C 语言类型             |
| ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- | ----------------------------- |
| [Int8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int8Array)                 | -128 到 127                                   | 1                  | 8 位有符号整数                                                                                                   | byte                                                      | int8_t                        |
| [Uint8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array)               | 0 到 255                                      | 1                  | 8 位无符号整数                                                                                                   | octet                                                     | uint8_t                       |
| [Uint8ClampedArray](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8ClampedArray) | 0 到 255                                      | 1                  | 8 位无符号整数（取值受控）                                                                                       | octet                                                     | uint8_t                       |
| [Int16Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int16Array)               | -32768 到 32767                               | 2                  | 16 位有符号整数                                                                                                  | short                                                     | int16_t                       |
| [Uint16Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint16Array)             | 0 到 65535                                    | 2                  | 16 位无符号整数                                                                                                  | unsigned short                                            | uint16_t                      |
| [Int32Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int32Array)               | -2147483648 到 2147483647                     | 4                  | 32 位有符号整数                                                                                                  | long                                                      | int32_t                       |
| [Uint32Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint32Array)             | 0 到 4294967295                               | 4                  | 32 位无符号整数                                                                                                  | unsigned long                                             | uint32_t                      |
| [Float32Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float32Array)           | 1.2x10<sup>-38</sup> 到 3.4x10<sup>38</sup>   | 4                  | 32 位单精度 IEEE 标准浮点数（[7 位有效数字](https://stackoverflow.com/a/50885022/14251417)例如： `1.123456`）    | unrestricted float                                        | float                         |
| [Float64Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float64Array)           | 5.0x10<sup>-324</sup> 到 1.8x10<sup>308</sup> | 8                  | 32 位双精度 IEEE 标准浮点数（[16 位有效数字](https://stackoverflow.com/a/50885022/14251417)例如： `1.123...15`） | unrestricted double                                       | double                        |
| [BigInt64Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt64Array)         | -2<sup>63</sup> 到 2<sup>63</sup>-1           | 8                  | 64 位有符号整数                                                                                                  | bigint                                                    | int64_t (signed long long)    |
| [BigUint64Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigUint64Array)       | 0 到 2<sup>64</sup>-1                         | 8                  | 54 位无符号整数                                                                                                  | bigint                                                    | uint64_t (unsigned long long) |

### DataView

[DataView](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView)提供了底层 getter/setter API，能够读写缓冲区数据。当处理不同类型数据时很有用。类型数组视图的操作遵循系统原生的字节序（[Endianness](https://developer.mozilla.org/en-US/docs/Glossary/Endianness)）进行存储。使用`DataView`则可以控制字节序，默认是大端，也可以使用 getter/setter 方法时设置成小端。

## 使用类型数组的一些 Web API

下面是一些使用到类型数组的 Web API，除此之外还有一些其他的，而且一直有更多 API 被加入进来：

[`FileReader.prototype.readAsArrayBuffer()`](<https://developer.mozilla.org/en-US/docs/Web/API/FileReader#readAsArrayBuffer()>)

`FileReader.prototype.readAsArrayBuffer()`方法可以读取指定的[Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob)或[File](https://developer.mozilla.org/en-US/docs/Web/API/File)内容。

[`XMLHttpRequest.prototype.send()`](<https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest#send()>)

`XMLHttpRequest`对象实例的`send()`方法现在支持类型数组，以[`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)对象作为参数。

[ImageData.data](https://developer.mozilla.org/en-US/docs/Web/API/ImageData)

`ImageData.data`是一个[`Uint8ClampedArray`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8ClampedArray)格式的一维数组，包含了 RGBA 顺序的整型数据，大小从 `0`到`255`（包含 `0` 和 `255`）

## 举例

### 使用 views 和 buffers

首先，我们需要创建一个 buffer，下面的 buffer 为固定的 16 字节长：

```js
let buffer = new ArrayBuffer(16);
```

现在，我们有一块初始值为 0 的内存。但是我们还不能使用它，不过我们可以确认其确实是 16 字节：

```js
if (buffer.byteLength === 16) {
  console.log("Yes, it's 16 bytes.");
} else {
  console.log("Oh no, it's the wrong size!");
}
```

为了真正使用它，我们需要创建一个 view。让我们创建一个 view，这个 view 会将 buffer 中的数据看做是 32 位符号整数组成的数组：

```js
let int32View = new Int32Array(buffer);
```

现在我们可以访问数组字段了，就像一个普通数组一样：

```js
for (let i = 0; i < int32View.length; i++) {
  int32View[i] = i * 2;
}
```

上述代码填充了数组中的 4 个条目（每个条目 4 个字节，一共 16 个字节），值依次为`0`, `2`, `4`, `6`。

### 同一份数据上不同的 views

当你在同一份数据上创建多份不同的 view 时会非常有趣。例如，接着上面的代码：

```js
let int16View = new Int16Array(buffer);

for (let i = 0; i < int16View.length; i++) {
  console.log("Entry " + i + ": " + int16View[i]);
}
```

我们接着创建了一个 16 位符号整数视图，它与先前的 32 位视图共享同一个 buffer，当我们以 16 位视图输出这个 buffer 中的值，结果为`0`, `0`, `2`, `0`, `4`, `0`, `6`, `0`。

> 译者注：[当前大部分计算机都是使用小端存储](https://developer.mozilla.org/en-US/docs/Glossary/Endianness)，因此最开始我们通过 32 位视图设置完成后，数据存储为：`0x00 0x00 0x00 0x00 0x02 0x00 0x00 0x00 ...`。当使用 16 位视图访问时，相当于`(0x00 0x00) (0x00 0x00) (0x02 0x00)...`，因此结果为`0`, `0`, `2`, `0`, `...`而不是`0`, `0`, `0`, `2`, `...`。

还可以更进一步，例如：

```js
int16View[0] = 32;
console.log("Entry 0 in the 32-bit array is now " + int32View[0]);
```

打印结果为`"Entry 0 in the 32-bit array is now 32"`。

换句话说，两个视图数组确实是访问同一个 buffer，将其视为不同格式。你还可以使用其他任意[类型数组](#类型数组视图)

### 构造更复杂的数据结构

通过对同一个 buffer 的数据使用不同视图，每个视图从 buffer 的不同偏移量开始，你可以创建并使用包含不同数据类型的数据结构对象。举个例子，你可以使用复杂的数据结构比如[WebGL](https://developer.mozilla.org/en-US/docs/Web/WebGL)、data files 或者当使用[`js-ctypes`](https://developer.mozilla.org/en-US/docs/Mozilla/js-ctypes)需要使用的 C 语言数据结构。

当有如下一个 C 语言数据结构：

```c
struct someStruct {
  unsigned long id;
  char username[16];
  float amountDue;
};
```

你可以像下面这样访问含有上述格式的 buffer 数据：

```js
let buffer = new ArrayBuffer(24);

// ... 将数据读进 buffer ...

let idView = new Uint32Array(buffer, 0, 1);
let usernameView = new Uint8Array(buffer, 4, 16);
let amountDueView = new Float32Array(buffer, 20, 1);
```

如果想知道`amountDue`的值，则可以通过`amountDueView[0]`获取。

> **注意**：C 语言中的[数据结构对齐](http://en.wikipedia.org/wiki/Data_structure_alignment)依赖于平台实现。对于结构的填充差异，请仔细考虑并采取一些预防措施。

### 转换为普通数组

创建类型数组后，有时候为了享受普通数组的一些便利方法，可以将其转换回普通数组。通过[`Array.from()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/from)可以将其转换回普通数组，如果不支持`Array.from()`，也可以使用下面代码：

```js
let typedArray = new Uint8Array([1, 2, 3, 4]),
  normalArray = Array.prototype.slice.call(typedArray);
normalArray.length === 4;
normalArray.constructor === Array;
```

## 规范

> [ECMAScript (ECMA-262)](https://tc39.es/ecma262/#sec-typedarray-objects)
> [The definition of 'TypedArray Objects' in that specification.](https://tc39.es/ecma262/#sec-typedarray-objects)

## 参考文章

- [Getting `ArrayBuffer`s or typed arrays from _Base64_-encoded strings](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Base64_encoding_and_decoding#Appendix.3A_Decode_a_Base64_string_to_Uint8Array_or_ArrayBuffer)
- [`StringView` – a C-like representation of strings based on typed arrays](https://developer.mozilla.org/en-US/docs/Code_snippets/StringView)
- [Faster Canvas Pixel Manipulation with Typed Arrays](https://hacks.mozilla.org/2011/12/faster-canvas-pixel-manipulation-with-typed-arrays)
- [Typed Arrays: Binary Data in the Browser](http://www.html5rocks.com/en/tutorials/webgl/typed_arrays)
- [Endianness](https://developer.mozilla.org/en-US/docs/Glossary/Endianness)
