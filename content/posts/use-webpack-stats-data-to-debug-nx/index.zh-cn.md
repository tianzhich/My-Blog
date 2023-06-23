---
title: "利用 webpack stats.json 定位 @nrwl/react webpack 配置问题"
date: 2020-12-06T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["Nx", "Webpack", "Debug", "Loader"]
categories: ["编程"]
featuredImage: "featured-image.webp"
---

利用 webpack stats.json 定位 @nrwl/react webpack 配置问题。

<!--more-->

## 背景

团队使用[NX](https://nx.dev/)这一 monorepo 工具来搭建 React 应用。NX 基于 React 应用在 webpack 打包时添加了[`url-loader`](https://github.com/nrwl/nx/blob/master/packages/react/plugins/webpack.ts)的相关配置。但是同事反馈该`url-loader`针对部分引用的图片文件不起作用。

## 定位

### url-loader 作用

`url-loader`，简而言之，可以将应用中引用到的一些资源文件（例如图片）转换成 base64 的数据格式，然后嵌入到我们的应用中（例如 HTML 的 img src, css 中的 url 函数），这样便无需针对该资源发起网络请求，节省请求资源。

以下是`url-loader`的配置举例：

```json
{
  "test": "/\\.(png|jpe?g|gif|webp)$/",
  "loader": require.resolve("url-loader"),
  "options": {
    "limit": 10000, // 10kB
    "name": "[name].[hash:7].[ext]"
  }
}
```

上述配置的意思就是针对 10kB 以下大小的一些常见图片格式文件使用`url-loader`处理，否则使用 fallback loader 处理，`url-loader`默认的 fallback loader 是`file-loader`。超过大小的文件会被其处理，相应的 options 会传给`file-loader`，例如 name，最终处理后的文件名会包括 7 位 hash。

### 具体针对哪部分文件不起作用？

由于 NX 并不是写好 webpack 配置，再使用 webpack 指令进行打包。而是以[Angular CLI builder](https://angular.io/guide/cli-builder)的形式，引入 webpack 并进行代码编写，拥有高度定制化，因此一开始并没有细看其 builder 的源码，很难找到具体是什么文件不起作用，而哪部分又没有问题。

一开始同事与之前的一个应用对比，发现其打包的一部分图片产物被`file-loader`处理，没有最终文件；而一部分则有最终文件，但是 hash 位数为默认的 20 位而不是`file-loader`处理后的 7 位；最后一部分文件则位数正确。

通过一些比对后发现，同一个图片文件，如果在样式文件（例如.scss）中引用则都会生成 20 位 hash 的文件名，而在 j(t)sx 中则配置生效。

### 查看 webpack config，尤其是样式文件那部分

找到对应的文件后，便需要查找是哪个 loader 处理了该文件，最先想到的是直接输出对应的`config.module`配置，由于 NX React 应用使用的配置文件为`@nrwl/react/plugins/webpack.js`，编辑该文件，增加如下部分：

PS: 由于正则表达式 JSON 没有对应的表达形式，因此 loader 的`test`部分只会是一个`{}`，不便于确认文件类型，因此可以使用其`toString()`方法作为`JSON.stringfy()`时的处理函数，以此直观显示匹配的文件类型。

```js
const fs = require("fs");

RegExp.prototype.toJSON = RegExp.prototype.toString;

function getWebpackConfig(config) {
  // ...
  fs.writeFile(
    "./webpack-config.json",
    JSON.stringify(config.module),
    null,
    () => {}
  );
  return config;
}

module.exports = getWebpackConfig;
```

打印出来的 config 如下（仅提取匹配部分）：

```json
{
  "rules": [
    {
      "test": "/\\.css$|\\.scss$|\\.sass$|\\.less$|\\.styl$/",
      "oneOf": [
        {
          "exclude": [
            "/Users/tianzhi/dev/nx-examples/libs/shared/styles/src/index.scss",
            "/Users/tianzhi/dev/nx-examples/libs/shared/header/index.scss",
            "/Users/tianzhi/dev/nx-examples/node_modules/normalize.css/normalize.css"
          ],
          "test": "/\\.scss$|\\.sass$/",
          "use": [
            {
              "loader": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/style-loader/dist/index.js"
            },
            {
              "loader": "/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js",
              "options": { "ident": "embedded", "sourceMap": false }
            },
            {
              "loader": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js",
              "options": {
                "implementation": {
                  "info": "dart-sass\t1.26.10\t(Sass Compiler)\t[Dart]\ndart2js\t2.8.4\t(Dart Compiler)\t[Dart]",
                  "types": {},
                  "NULL": {},
                  "TRUE": { "value": true },
                  "FALSE": { "value": false }
                },
                "sourceMap": false,
                "sassOptions": { "precision": 8, "includePaths": [] }
              }
            }
          ]
        }
      ]
    },
    {
      "test": "/\\.(png|jpe?g|gif|webp)$/",
      "loader": "/Users/tianzhi/dev/nx-examples/node_modules/url-loader/dist/cjs.js",
      "options": { "limit": 10000, "name": "[name].[hash:7].[ext]" }
    },
    {
      "test": "/\\.svg$/",
      "oneOf": [
        {
          "issuer": { "test": "/\\.[jt]sx?$/" },
          "use": [
            "@svgr/webpack?-svgo,+titleProp,+ref![path]",
            {
              "loader": "/Users/tianzhi/dev/nx-examples/node_modules/url-loader/dist/cjs.js",
              "options": {
                "limit": 10000,
                "name": "[name].[hash:7].[ext]",
                "esModule": false
              }
            }
          ]
        },
        {
          "use": [
            {
              "loader": "/Users/tianzhi/dev/nx-examples/node_modules/url-loader/dist/cjs.js",
              "options": { "limit": 10000, "name": "[name].[hash:7].[ext]" }
            }
          ]
        }
      ]
    }
  ]
}
```

可以看到针对.scss 文件处理的 loader 及顺序为：`sass-loader` -> `postcss-loader` -> `style-loader`。

最疑惑的地方也就在这，一般情况下，我们使用了`postcss-loader`后还会使用`css-loader`进行处理，`css-loader`会解析`@import`以及`url()`语法，将其转化为`import/require()`，这样一来便能交给其他 loader 例如`url-loader`进行处理。而对于`postcss-loader`，是因为它的[autoprefixer](https://github.com/postcss/autoprefixer)非常出名。

所以到此为止还是无法确认具体是哪个 loader 处理了`url()`语法以至于`url-loader`无法进行处理。但是可以确定的是答案就在这三个 loader 之一，由于这个时候还不了解`postcss-loader`的插件机制，我甚至怀疑是`sass-loader`的某个配置使得`url()`语法被提前解析。但是猜测始终不是办法，需要找到一个更确定的方法，能够知道对应文件在 loader 处理前后的产物文件。

### 使用 stats.json 定位问题

如果能 debugging webpack 打包这一过程就好了，查找官网，还真发现有针对[webpack 打包过程的 debug 方法](https://webpack.js.org/contribute/debugging/)，有两种方案，我选用了较为简单的日志方案：查看数据报告 [stats.json 文件](https://webpack.js.org/api/stats/)

webpack 使用`--json`指令可以输出对应文件，在 NX 中则是[`--statsJson`](https://nx.dev/latest/react/cli/build#statsjson)

输出后的日志报告条目很多，关于更多条目可以参考[官网介绍](https://webpack.js.org/api/stats/)，这里只需要搜索对应文件，例如我这里是`app.scss`，会得到如下信息（仅提取有效信息）：

```json
{
  "chunks": [
    {
      "names": ["main"],
      "files": ["main.48c162c6863d37cba541.es5.js"],
      "hash": "48c162c6863d37cba541",
      "modules": [
        {
          "id": "/CXp",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/style-loader/dist/index.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??embedded!/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "name": "./app/app.scss",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
          "issuerId": null,
          "issuerName": "./app/app.tsx",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
              "name": "./app/app.tsx"
            }
          ],
          "source": "var content = require(\"!!../../../../node_modules/postcss-loader/src/index.js??embedded!../../../../node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!./app.scss\");\n\nif (typeof content === 'string') {\n  content = [[module.id, content, '']];\n}\n\nvar options = {}\n\noptions.insert = \"head\";\noptions.singleton = false;\n\nvar update = require(\"!../../../../node_modules/@nrwl/web/node_modules/style-loader/dist/runtime/injectStylesIntoStyleTag.js\")(content, options);\n\nif (content.locals) {\n  module.exports = content.locals;\n}\n"
        },
        {
          "id": "GAnJ",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/style-loader/dist/runtime/injectStylesIntoStyleTag.js",
          "name": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/style-loader/dist/runtime/injectStylesIntoStyleTag.js",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/style-loader/dist/index.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??embedded!/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "issuerId": "/CXp",
          "issuerName": "./app/app.scss",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
              "name": "./app/app.tsx"
            },
            {
              "id": "/CXp",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/style-loader/dist/index.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??embedded!/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "./app/app.scss"
            }
          ],
          "source": "..."
        },
        {
          "id": null,
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
          "name": "./app/app.tsx",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
          "issuerId": null,
          "issuerName": "./main.tsx",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            }
          ],
          "source": "... import './app.scss';\n ..."
        },
        {
          "id": "aZ7I",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??embedded!/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "name": "/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src??embedded!/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!./app/app.scss",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/style-loader/dist/index.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??embedded!/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "issuerId": "/CXp",
          "issuerName": "./app/app.scss",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
              "name": "./app/app.tsx"
            },
            {
              "id": "/CXp",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/style-loader/dist/index.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??embedded!/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "./app/app.scss"
            }
          ],
          "assets": [
            "banner2.50acf29cb9b5f2021724.png",
            "arrow-right.d9910f497ca75d78bcee.svg",
            "add-big@2x.ebff90ff08575204ca08.png"
          ],
          "source": "module.exports = \".image-png{background-image:url('add-big@2x.ebff90ff08575204ca08.png')}.image-svg{background-image:url('arrow-right.d9910f497ca75d78bcee.svg')}.image-big-png{background-image:url('banner2.50acf29cb9b5f2021724.png')}\""
        }
      ]
    }
  ]
}
```

先来说几个条目概念：

1. `chunks`是我们这次打包的 chunk 列表，每个 chunk 里的`modules`对应组成该 chunk 的 module 列表
2. `identifier`为模块内部唯一标识
3. `issuer`为该模块的引用来源
4. `source`为模块的 stringfy 后的源码
5. `assets`为该模块包含的静态资源

依照`issuer`，我们可以得到调用顺序为：

- `app.tsx`
  - `/CXp(style-loader!postcss-loader!sass-loader!app.scss)`
    - `GAnJ(style-loader/injectStylesIntoStyleTag.js)`
    - `aZ7I(postcss-loader!sass-loader!app.scss)`

可以理解为：`app.tsx`中`import './app.scss'`被匹配并解析为内部模块`/CXp`，该模块源码`source`format 后为：

```js
var content = require("!!../../../../node_modules/postcss-loader/src/index.js??embedded!../../../../node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!./app.scss");

if (typeof content === "string") {
  content = [[module.id, content, ""]];
}

var options = {};

options.insert = "head";
options.singleton = false;

var update =
  require("!../../../../node_modules/@nrwl/web/node_modules/style-loader/dist/runtime/injectStylesIntoStyleTag.js")(
    content,
    options
  );

if (content.locals) {
  module.exports = content.locals;
}
```

不难发现，该模块又引用了内部模块`aZ7I`和第三方`style-loader`的模块`GAnJ`

第三方`style-loader`的模块`GAnJ`，也就是`injectStylesIntoStyleTag.js`的作用正如名字所述，会将最终样式产物插入到 HTML 的 style 标签。我们只需要继续分析生成的内部模块`aZ7I`，其`source`如下：

```js
module.exports =
  ".image-png{background-image:url('add-big@2x.ebff90ff08575204ca08.png')}.image-svg{background-image:url('arrow-right.d9910f497ca75d78bcee.svg')}.image-big-png{background-image:url('banner2.50acf29cb9b5f2021724.png')}";
```

没有再次引用，仅为最终样式的字符串，将其提取为 CSS：

```css
.image-png {
  background-image: url("add-big@2x.ebff90ff08575204ca08.png");
}
.image-svg {
  background-image: url("arrow-right.d9910f497ca75d78bcee.svg");
}
.image-big-png {
  background-image: url("banner2.50acf29cb9b5f2021724.png");
}
```

就是源码中的三个`url()`调用，但是里面的路径已经被解析，引用文件名包含 webpack 默认的 20 位 hash 而不是`url-loader`配置中的 7 位，而且前两个图片文件大小均小于 10KB，本应该被`url-loader`转换为 base64 格式。

看到这里，其实仅仅知道`url()`被经过模块`aZ7I`后被更改，`aZ7I`的标识字段为`/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??embedded!/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/node_modules/sass-loader/dist/cjs.js??ref--5-oneOf-3-2!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss`。

将标识以`!`分隔后，得到`.../postcss-loader/src/index.js??embedded`，`.../sass-loader/dist/cjs.js??ref--5-oneOf-3-2`以及`app.scss`，从标识符中我们可以知道 webpack 处理 `app.scss`文件使用的 loader 及顺序，即经过`sass-loader`处理后再由`postcss-loader`处理形成了上述产物。

这里我有一个疑问：

> 为什么 webpack 没有把过程继续拆分，也就是把上述模块再细化成两个模块，多出的一个模块则单独为`sass-loader`处理后的产物。如果能够细分，则能知道具体是哪个 loader 处理了`url()`

我暂时没找到这个问题的答案，所以继续查找。

先是去搜索`sass-loader`，发现其[并没有处理`url()`](https://webpack.js.org/loaders/sass-loader/#problems-with-url)；然后是`postcss-loader`，发现了其支持第三方插件，在这之前我还提了一个[issue](https://github.com/webpack-contrib/postcss-loader/issues/500)，通过逐步寻找，发现[`postcss-url插件`](https://github.com/postcss/postcss-url)会处理`url()`，但是我在 NX 的依赖中却没找到它。

知道插件机制后，我同时也开始阅读 NX 源码的 style loader 配置部分，终于发现原来是它们自己创建了一个插件用来处理`@import`和`url()`，相当于替代了`css-loader`。但是针对 react 的 webpack 配置中引入`url-laoder`时却没有考虑这部分，导致图片等文件无法再被`url-loader`处理。

## 解决方案

尽管这个问题的影响没有那么大，但是一定要避免对同一个大于 10KB 的图片文件同时在样式文件和 j(t)sx 文件中引用，这样会导致同一个文件产生两份产出，一个文件名 7 位 hash，另一个则是 20 位 hash，这会导致针对同一个文件发起两次网络请求，如果文件较多将会浪费很多资源。而且这确实也会让人费解，因此我提出了一个[issue](https://github.com/nrwl/nx/issues/4122)用于追踪。

但是鉴于 NX 关于 style 部分的 webpack 配置是集成在定制化的 webpack builder 代码文件中，而`url-loader`等配置却是 react 单独的，估计 NX 也不好修改这部分。

给团队的临时解决方案是：

1. 如果想使用`url-loader`处理所有图片文件，尽量使用`<img />`标签而不是`url()`来引入图片
2. 如果并不一定要使用`url-loader`处理，可以使用绝对路径前缀例如`/assets/xxx`引用，同时配置[`assets`选项](https://nx.dev/latest/react/cli/build#assets)，个人认为，这才是`assets`配置的正确使用方式
3. 最后还是一定要避免对同一个大于 10KB 的图片文件同时在样式文件和 j(t)sx 文件中引用

## 最后的探索

给出方案后，问题其实可以被较好地解决，但是我发现 NX 使用`postcss-loader`，除了最常见的 autoprefixer，还做的两件事就是使用社区的[`postcss-import`](https://github.com/postcss/postcss-import)来处理`@import`，以及使用自己写的[`postcss-cli-resources`](https://github.com/nrwl/nx/blob/master/packages/web/src/utils/third-party/cli-files/plugins/postcss-cli-resources.ts)来处理`url()`。

其实社区处理`url()`也有一个插件[`postcss-url`](https://github.com/postcss/postcss-url)，暂时无法得知为什么 NX 需要自己创建一个插件来处理。

做的这三件事中，`url()`和`@import`也可以交给`css-loader`处理，**不过 NX 传入了一些路径参数，如果这样替换，我们便无法正常使用这些参数，例如同时开启[`rebaseRootRelativeCssUrls`](https://nx.dev/latest/react/cli/build#rebaserootrelativecssurls)和[`deployUrl`](https://nx.dev/latest/react/cli/build#deployurl)后，NX 会在打包时将`deployUrl`作为`url()`声明路径的前缀。**

但是为了继续探索`url-loader`正常工作下这些文件解析的产物和顺序，我还是对配置进行了一些更改，主要包括覆盖官方的`postcss-loader`配置，仅使用其 autoprefixer 功能，以及在其后添加`css-loader`，完成后输出的`stats.json`如下（仅提取有效信息）：

```json
{
  "chunks": [
    {
      "names": ["main"],
      "files": ["main.413a212ab2c713a5f9c4.es5.js"],
      "hash": "413a212ab2c713a5f9c4",
      "modules": [
        {
          "id": "/CXp",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/style-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "name": "./app/app.scss",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
          "issuerId": null,
          "issuerName": "./app/app.tsx"
        },
        {
          "id": "JDPv",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/url-loader/dist/cjs.js??ref--7-oneOf-1-0!/Users/tianzhi/dev/nx-examples/apps/cart/src/assets/arrow-right.svg",
          "name": "./assets/arrow-right.svg",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "issuerId": "V1gc",
          "issuerName": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!./app/app.scss",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
              "name": "./app/app.tsx"
            },
            {
              "id": "/CXp",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/style-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "./app/app.scss"
            },
            {
              "id": "V1gc",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!./app/app.scss"
            }
          ],
          "source": "export default \"data:image/svg+xml;base64,...\""
        },
        {
          "id": "LPAU",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/style-loader/dist/runtime/injectStylesIntoStyleTag.js",
          "name": "/Users/tianzhi/dev/nx-examples/node_modules/style-loader/dist/runtime/injectStylesIntoStyleTag.js",
          "issuerId": "/CXp",
          "issuerName": "./app/app.scss"
        },
        {
          "id": null,
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
          "name": "./app/app.tsx",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx"
        },
        {
          "id": "V1gc",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "name": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!./app/app.scss",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/style-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "issuerId": "/CXp",
          "issuerName": "./app/app.scss",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
              "name": "./app/app.tsx"
            },
            {
              "id": "/CXp",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/style-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "./app/app.scss"
            }
          ],
          "source": "// Imports\nimport ___CSS_LOADER_API_IMPORT___ from \"../../../../node_modules/css-loader/dist/runtime/api.js\";\nimport ___CSS_LOADER_GET_URL_IMPORT___ from \"../../../../node_modules/css-loader/dist/runtime/getUrl.js\";\nimport ___CSS_LOADER_URL_IMPORT_0___ from \"../assets/add-big@2x.png\";\nimport ___CSS_LOADER_URL_IMPORT_1___ from \"../assets/arrow-right.svg\";\nimport ___CSS_LOADER_URL_IMPORT_2___ from \"../assets/banner2.png\";\nvar ___CSS_LOADER_EXPORT___ = ___CSS_LOADER_API_IMPORT___(true);\nvar ___CSS_LOADER_URL_REPLACEMENT_0___ = ___CSS_LOADER_GET_URL_IMPORT___(___CSS_LOADER_URL_IMPORT_0___);\nvar ___CSS_LOADER_URL_REPLACEMENT_1___ = ___CSS_LOADER_GET_URL_IMPORT___(___CSS_LOADER_URL_IMPORT_1___);\nvar ___CSS_LOADER_URL_REPLACEMENT_2___ = ___CSS_LOADER_GET_URL_IMPORT___(___CSS_LOADER_URL_IMPORT_2___);\n// Module\n___CSS_LOADER_EXPORT___.push([module.id, \".image-png{background-image:url(\" + ___CSS_LOADER_URL_REPLACEMENT_0___ + \")}.image-svg{background-image:url(\" + ___CSS_LOADER_URL_REPLACEMENT_1___ + \")}.image-big-png{background-image:url(\" + ___CSS_LOADER_URL_REPLACEMENT_2___ + \")}.example>.example-inner{font-size:14px}\", \"\",{\"version\":3,\"sources\":[\"webpack://app/app.scss\"],\"names\":[],\"mappings\":\"AAAA,WAAW,wDAAgD,CAAC,WAAW,wDAAiD,CAAC,eAAe,wDAA6C,CAAC,wBAAwB,cAAc\",\"sourcesContent\":[\".image-png{background-image:url(\\\"../assets/add-big@2x.png\\\")}.image-svg{background-image:url(\\\"../assets/arrow-right.svg\\\")}.image-big-png{background-image:url(\\\"../assets/banner2.png\\\")}.example>.example-inner{font-size:14px}\"],\"sourceRoot\":\"\"}]);\n// Exports\nexport default ___CSS_LOADER_EXPORT___;\n"
        },
        {
          "id": "VNgF",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/runtime/api.js",
          "name": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/runtime/api.js",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "issuerId": "V1gc",
          "issuerName": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!./app/app.scss",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
              "name": "./app/app.tsx"
            },
            {
              "id": "/CXp",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/style-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "./app/app.scss"
            },
            {
              "id": "V1gc",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!./app/app.scss"
            }
          ],
          "source": "..."
        },
        {
          "id": "m1aJ",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/url-loader/dist/cjs.js??ref--6!/Users/tianzhi/dev/nx-examples/apps/cart/src/assets/banner2.png",
          "name": "./assets/banner2.png",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
          "issuerId": null,
          "issuerName": "./app/app.tsx",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
              "name": "./app/app.tsx"
            }
          ],
          "source": "export default __webpack_public_path__ + \"banner2.e11abd9.png\";"
        },
        {
          "id": "q/iR",
          "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/runtime/getUrl.js",
          "name": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/runtime/getUrl.js",
          "issuer": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
          "issuerId": "V1gc",
          "issuerName": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!./app/app.scss",
          "issuerPath": [
            {
              "id": 0,
              "identifier": "multi /Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "multi ./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/main.tsx",
              "name": "./main.tsx"
            },
            {
              "id": null,
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/@nrwl/web/src/utils/web-babel-loader.js??ref--4!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.tsx",
              "name": "./app/app.tsx"
            },
            {
              "id": "/CXp",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/style-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "./app/app.scss"
            },
            {
              "id": "V1gc",
              "identifier": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src/index.js??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/apps/cart/src/app/app.scss",
              "name": "/Users/tianzhi/dev/nx-examples/node_modules/css-loader/dist/cjs.js!/Users/tianzhi/dev/nx-examples/node_modules/postcss-loader/src??ref--8-2!/Users/tianzhi/dev/nx-examples/node_modules/sass-loader/dist/cjs.js!./app/app.scss"
            }
          ],
          "source": "..."
        }
      ]
    }
  ]
}
```

解读之前，先来看`app.scss`源文件和`app.tsx`文件（仅提取引用信息）：

```scss
$font: 14px;

.image-svg {
  background-image: url("../assets/arrow-right.svg");
}
.image-big-png {
  background-image: url("../assets/banner2.png");
}

.example {
  & > .example-inner {
    font-size: $font;
  }
}
```

```tsx
import "./app.scss";
import banner from "../assets/banner2.png";
import arrow from "../assets/arrow-right.svg";
```

其中，`arrow-right.svg`体积小于 10KB，而`banner2.png`大于 10KB。

解析顺序为：

- `app.tsx`
  - `m1aJ(banner2.png)`
  - `/CXp(style-loader!css-loader!postcss-loader!sass-loader!app.scss)`
    - `LPAU(style-loader/injectStylesIntoStyleTag.js)`
    - `V1gc(css-loader!postcss-loader!sass-loader!app.scss)`
      - `JDPv(arrow-right.svg)`
      - `VNgF(css-loader/api.js)`
      - `q/iR(css-loader/getUrl.js)`

可以看到，`banner2.png`先在`app.tsx`中被解析，而`arrow-right.svg`却在`app.scss`中被解析，关于原因，暂时也没有深入研究，如果有理解的同学欢迎留言。

跟前面一样，`import './app.scss'`被匹配解析为内部模块`/CXp(style-loader!css-loader!postcss-loader!sass-loader!app.scss)`，其`source`format 后和前面一致，唯一不同的是引用的子模块多了一个`css-loader`，对子模块`V1gc(css-loader!postcss-loader!sass-loader!app.scss)`的`source`进行 format 后得到：

```js
// Imports
import ___CSS_LOADER_API_IMPORT___ from "../../../../node_modules/css-loader/dist/runtime/api.js";
import ___CSS_LOADER_GET_URL_IMPORT___ from "../../../../node_modules/css-loader/dist/runtime/getUrl.js";
import ___CSS_LOADER_URL_IMPORT_0___ from "../assets/arrow-right.svg";
import ___CSS_LOADER_URL_IMPORT_1___ from "../assets/banner2.png";
var ___CSS_LOADER_EXPORT___ = ___CSS_LOADER_API_IMPORT___(true);
var ___CSS_LOADER_URL_REPLACEMENT_0___ = ___CSS_LOADER_GET_URL_IMPORT___(
  ___CSS_LOADER_URL_IMPORT_0___
);
var ___CSS_LOADER_URL_REPLACEMENT_1___ = ___CSS_LOADER_GET_URL_IMPORT___(
  ___CSS_LOADER_URL_IMPORT_1___
);
// Module
___CSS_LOADER_EXPORT___.push([
  module.id,
  ".image-svg{background-image:url(" +
    ___CSS_LOADER_URL_REPLACEMENT_0___ +
    ")}.image-big-png{background-image:url(" +
    ___CSS_LOADER_URL_REPLACEMENT_1___ +
    ")}.example>.example-inner{font-size:14px}",
  "",
  {
    version: 3,
    sources: ["webpack://app/app.scss"],
    names: [],
    mappings:
      "AAAA,WAAW,wDAAgD,CAAC,WAAW,wDAAiD,CAAC,eAAe,wDAA6C,CAAC,wBAAwB,cAAc",
    sourcesContent: [
      '.image-svg{background-image:url("../assets/arrow-right.svg")}.image-big-png{background-image:url("../assets/banner2.png")}.example>.example-inner{font-size:14px}',
    ],
    sourceRoot: "",
  },
]);
// Exports
export default ___CSS_LOADER_EXPORT___;
```

可以清晰看到，除了引用两个自身 runtime 模块外，还引用了两个图片，由于`banner2.png`事先已被解析，因此这里只需要解析`arrow-right.svg`

拆分`banner2.png`和`arrow-right.svg`的`identifier`可以得出它们解析时使用的 loader：

1. `banner2.png`的`identifier`为：`/Users/tianzhi/dev/nx-examples/node_modules/url-loader/dist/cjs.js??ref--6!/Users/tianzhi/dev/nx-examples/apps/cart/src/assets/banner2.png`

2. `arrow-right.svg`的`identifier`为：`/Users/tianzhi/dev/nx-examples/node_modules/url-loader/dist/cjs.js??ref--7-oneOf-1-0!/Users/tianzhi/dev/nx-examples/apps/cart/src/assets/arrow-right.svg`

两者都使用了`url-loader`进行处理，不过要注意的是，`banner2.png`其实使用的是`url-loader`的默认 fallback loader 也就是`file-loader`处理。可以看到`banner2.png`模块的`source`为：

> `export default __webpack_public_path__ + \"banner2.e11abd9.png\"`;

而`arrow-right.svg`模块则为：

> `export default \"data:image/svg+xml;base64,...\"`

## 总结

这次踩坑之旅总体来说效率不算高，花了一些时间，主要是因为自己不熟悉`postcss-loader`的插件机制以及 NX 的源码结构。而且最后也遗留了几个问题，只算是对 webpack debugging 的一次初级入门，希望今后能掌握第二种[DevTools](https://webpack.js.org/contribute/debugging/#devtools)的方式，同时解决这些遗留问题。
