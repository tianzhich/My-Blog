---
title: "使用 Node 工具简化文件创建流程"
description: "React中创建组件，添加Redux，添加Route，(或者还需要添加saga)，最后在全局Route和Redux中加入它们，这一系列重复的过程，都可以在Node中创建'脚本'来自动化执行。"
date: 2019-01-27T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["NodeJS"]
categories: ["编程"]
featuredImage: "featured-image.webp"
---

React 中创建组件，添加 Redux，添加 Route，(或者还需要添加 saga)，最后在全局 Route 和 Redux 中加入它们，这一系列重复的过程，都可以在 Node 中创建脚本来自动化执行。

<!--more-->

## 背景

最近搭建毕设前端框架的时候，每当创建一个页面，我都需要创建这个页面组件，创建它的 Route，最后将该 Route 加入到总的 Route。当然，这里的流程还不算复杂，基本也是复制粘贴改改变量，但是后面还要用到 Redux，可能还会使用 saga…再将它们加入这一条流程线，我需要改的东西又多了

在公司实习的脚手架里，发现有大佬造的轮子，之前也只是照着命令敲拿来用，这次顺带研究了一下核心功能，结合我的毕设框架需要，加入了最简单的自动化“脚本”。可能是我的搜索方式有问题，没在网上找到类似的，也因此把它记录下来 📝

## 所需环境和工具

1. Node 环境下执行

2. 命令映射，使用[commander](https://github.com/tj/commander.js)

   > 让文件可以通过命令行的形式执行

3. 文件读写，这里我使用的是[fs-extra](https://github.com/jprichardson/node-fs-extra)，使用 Node 自带的[File System](https://nodejs.org/api/fs.html)，但是前者支持 Promise 和 Async, Await

   > 文件读写只是读取模板文件内容，然后写入到新的文件为我们所用

4. 模板字符串，使用 lodash/string 的模板字符串方法[template](https://lodash.com/docs/4.17.10#template)

   > 模板字符串：我们可以使用 xxx.tmpl 格式的文件存储我们的模板，需要替换的内容使用
   >
   > `<%= xxx %>`表示即可，下面会给出文件原型

5. 文件修改，使用[ts-simple-ast](https://github.com/dsherret/ts-simple-ast)

   > 文件修改则是直接修改原来文件，加入自己所需的东西，例如修改变量值，这也是这篇文章中提到的较为简单的一个用途，其他更复杂的也可以参考文档学习

## 文件原型和需求

### 需求

每当新增一个页面，我们需要创建一个基本框架组件，一个 Route，最后把这个 Route 自动插入到总的 Router 里

### xxxComponent

这里创建了一个非常简单的组件，带有 Props 和 State，interface 使用 Ixxx 命名

```tsx
import React from 'react';

interface I<%= featureUpperName %>Props {}

interface I<%= featureUpperName %>State {}

export default class <%= featureUpperName %> extends React.Component<I<%= featureUpperName %>Props, I<%= featureUpperName %>State> {
    constructor(props: I<%= featureUpperName %>Props) {
        super(props);
        this.state = {};
    }

    render() {
        return (
            <h2>My Home</h2>
        )
    }
}
```

### xxx.index

这个文件里加入所有需要导出的 Component，并作为统一导出出口

```ts
export { default as <%= featureUpperName %> } from './<%= featureUpperName %>'
```

### Route

自定义的 Route，属性也基本遵循原生 Route，加入 loadable component，支持按需加载

```ts
import App from "../common/component/App";
import { IRoute } from "@src/common/routeConfig";

const loader = (name: string) => async () => {
  const entrance = await import("./");
  return entrance[name];
};

const childRoutes: IRoute[] = [
  {
    path: "/<%= featureName %>",
    name: "<%= featureUpperName %>",
    loader: loader("<%= featureUpperName %>"),
  },
];

export default {
  path: "/<%= featureName %>",
  name: "",
  component: App,
  childRoutes,
};
```

**上面三个便作为基本模板文件，下面这个则是总的 Route**

### routeConfig

> 完成一个页面的创建并生成它的 route 后，需要在该文件引入这个 route，然后修改变量 childRoutes，插入该 route，这样我们的工作就算完成啦

```ts
import HomeRoute from "../features/home/route";

export interface IRoute {
  path: string;
  name: string;
  component?: React.ComponentClass | React.SFC;
  childRoutes?: IRoute[];
  loader?: AsyncComponentLoader;
  exact?: boolean;
  redirect?: string;
}

const childRoutes: IRoute[] = [HomeRoute];

const routes = [
  {
    path: "/",
    name: "app",
    exact: true,
    redirect: "/home",
  },
  ...childRoutes,
];

export default routes;
```

## 步骤

### 源代码

#### kit.js

**用于读取模板文件，写入新的文件**

首先第一行，告诉 shell 此文件默认执行环境为 Node

接下来我们来看 addFeatureItem(忽略我的命名 ╮(╯▽╰)╭)，这个函数有三个参数

- srcPath，template 文件位置
- targetPath，写入的文件位置
- option，渲染模板时使用，简而言之可以替换掉模板中的变量为里面我们设定的值

我们先确认文件是否存在，然后读取模板文件，写入新的文件即可，中间加了个已有文件判断

是不是很简单！

最后加入使用 commander 创建自己的命令即可，更详细的用法可以查看 commander 的[文档](https://github.com/tj/commander.js)，这里添加一个简单的 add 命令，后跟一个 featureName，键入命令后执行 action 函数，里面的参数即我们刚刚键入的 featureName，读取后便可以从模板创建新的 feature

当然，我们还需要修改 routeConfig.ts 这个文件，我将这个操作放到了下面的 ts-ast.ts 文件

```js
#! /usr/bin/env node

const fse = require("fs-extra");
const path = require("path");
const _ = require("lodash/string");
const commander = require("commander");
const ast = require("./ts-ast");

const templatesDir = path.join(__dirname, "templates");
const targetDir = path.join(__dirname, "..", "src", "features");

async function addFeatureItem(srcPath, targetPath, option) {
  let res;
  try {
    await fse.ensureFile(srcPath);
    res = await fse.readFile(srcPath, "utf-8");

    // avoid override
    const exists = await fse.pathExists(targetPath);
    if (exists) {
      console.log(`${targetPath} is already added!`);
      return;
    }

    await fse.outputFile(targetPath, _.template(res)(option), {
      encoding: "utf-8",
    });
    console.log(`Add ${srcPath} success!`);
  } catch (err) {
    console.error(err);
  }
}

async function addFeature(name) {
  const renderOpt = {
    featureName: name,
    featureUpperName: _.upperFirst(name),
  };

  const componentTmpl = `${templatesDir}/Component.tsx.tmpl`;
  const component = `${targetDir}/${name}/${_.upperFirst(name)}.tsx`;
  addFeatureItem(componentTmpl, component, renderOpt);

  const indexTmpl = `${templatesDir}/index.ts.tmpl`;
  const index = `${targetDir}/${name}/index.ts`;
  addFeatureItem(indexTmpl, index, renderOpt);

  const routeTmpl = `${templatesDir}/route.ts.tmpl`;
  const route = `${targetDir}/${name}/route.ts`;
  addFeatureItem(routeTmpl, route, renderOpt);
}

commander
  .version(require("../package.json").version)
  .command("add <feature>")
  .action((featureName) => {
    // add features
    addFeature(featureName);
    // manipulate some ts file like route
    ast(featureName);
  });

commander.parse(process.argv);
```

#### ts-ast.js

**用于修改 rootConfig.ts 文件**

先给出 ts-simple-ast 的[地址](https://github.com/dsherret/ts-simple-ast)，自己还是觉得这个操作是比较复杂的，我也是参考了文档再加上项目脚手架代码才看明白，至于原理性的东西，可能还需要查看[Typescript Compiler API](https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API)，因为这个包也只是 Wrapper，文档也还不是很完善，更复杂的需求还有待学习

这里关键就两个操作，一个是添加一个 import，其次则是修改 childRoutes 变量的值。但是一些函数的英文字面意思理解起来可能比上面的文件读写要困难

1. 我们首先需要新建一个 Project
2. 然后需要获取待操作的文件，拿到待操作文件(srcFile)之后，直接使用*addImportDeclaration*这个方法便可以添加 default import，如果需要 named import，也可以使用 addNamedImport。定义好 default name(defaultImport prop)以及 path(moduleSpecifier)即可
3. 最后是对 childRoutes 变量的值进行修改，这一过程比较复杂
   - 我们首先需要通过变量名拿到一个 VariableStatement
   - 然后再拿到变量声明（VariableDeclaration），因为一个文件中可能有多处声明（函数中，全局中）
   - 由于该例中只有一处声明，所以我们直接 forEach 遍历声明即可，遍历时拿到 initializer（这里是一个类似 array 的东东）
   - 再使用它的 forEachChild 遍历拿到 node，node.getText 终于拿到了里面的一个值（比如 HomeRoute）
   - 我们将这些值添加到一个新的数组
   - 直到遍历完毕，再将它们拼接起来，加入新的 route，以字符串形式 setInitializer 即可
4. 最后保存所有操作，Project.save()即可

```js
const { Project } = require("ts-simple-ast");
const _ = require("lodash/string");
const path = require("path");

const project = new Project();
const srcDir = path.join(__dirname, "..", "src", "common");

async function addRoute(featureName) {
  const FeatureName = _.upperFirst(featureName);
  try {
    const srcFile = project.addExistingSourceFile(`${srcDir}/routeConfig.ts`);
    srcFile.addImportDeclaration({
      defaultImport: `${FeatureName}Route`,
      moduleSpecifier: `../features/${featureName}/route`,
    });
    const routeVar = srcFile.getVariableStatementOrThrow("childRoutes");

    let newRoutes = [];

    routeVar.getDeclarations().forEach((decl, i) => {
      decl.getInitializer().forEachChild((node) => {
        newRoutes.push(node.getText());
      });
      decl.setInitializer(`[${newRoutes.join(", ")}, ${FeatureName}Route]`);
    });

    await project.save();
    console.log("Add route successful");
  } catch (err) {
    console.log(err);
  }
}

module.exports = addRoute;
```

### 修改文件权限，使其可执行

只要头部加入了`#! /usr/bin/env node`，简单的一行命令即可搞定`chmod +x /filePath/yourExeFile`

然后，我们便可以使用`/filePath/yourExeFile add featureName`的方式添加一个新的页面！

## 参考

1. [what-does-chmod-x-filename-do-and-how-do-i-use-it](https://askubuntu.com/questions/443789/what-does-chmod-x-filename-do-and-how-do-i-use-it/443799)
