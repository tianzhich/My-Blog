---
title: "【Rematch 源码系列】四、Third-Party plugins"
date: 2021-08-11T20:43:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["rematch", "redux", "源代码"]
categories: ["编程"]
series: ["rematch-code"]
series_weight: 5
featuredImage: "featured-image.webp"
---

Rematch 源码解读系列的第 4️⃣ 篇，关于 Rematch 的第三方插件。

<!--more-->

> 如无特殊说明，本专栏文章的代码版本均为 @rematch/core: 1.4.0

[上篇](../rematch-series-plugin-factory-and-core-plugins/)介绍了 rematch 的插件机制以及其核心的两个插件。除了这两个插件外，rematch 团队其实还开发了不少第三方插件，例如 immer，loading，select，persist，updated 等等。这篇文章我们选取两个使用最多的插件来介绍，它们分别是 immer 和 loading。

在讲解之前，还是先回顾一下 rematch 的代码结构和组成部分：

```
...
plugins
|—— ...
|—— loading
|—— immer
|—— select
src
|—— plugins
|  |—— dispatch.ts
|  |—— effects.ts
|—— typings
|  |—— index.ts
|—— utils
|  |—— deprecate.ts
|  |—— isListener.ts
|  |—— mergeConfig.ts
|  |—— validate.ts
|—— index.ts
|—— pluginFactory.ts
|—— redux.ts
|—— rematch.ts
```

可以看到，上面提到的两个核心 plugin 是放在 `src` 目录下的，而第三方插件位于一个单独的 `plugins` 目录下。后面的文章会提到，在 Rematch v2 中，我们将这两个核心 plugin 集成到了 `rematch/core`，所以将不再有 `src/plugins` 目录。

下面为 rematch 的组成部分：

![rematch 组成部分](./rematch-code-structure.png)

## immer

immer 插件的灵感来源于 [immerjs](https://github.com/immerjs/immer)，方便你在 reducer 中以 mutable 的方式生成 immutable 的数据。简而言之，如果没有 immer，我们在 reducer 中通常的做法是：

```ts
function myReducer(state, action) {
  return {
    ...state,
    slice: action.payload,
  };
}
```

而使用 immer 后，我们可以：

```ts
function myReducer(state, action) {
  state.slice = action.payload;
}
```

是不是简单了许多，要实现 immer plugin 比较简单，只需要给正常的 reducer 包一层，使其经过 immer 处理即可，我们来看看代码实现：

```ts
function combineReducersWithImmer(reducers: ReducersMapObject) {
  const reducersWithImmer: ReducersMapObject<any, Action<any>> = {};
  // reducer must return value because literal don't support immer
  for (const key of Object.keys(reducers)) {
    const reducerFn = reducers[key];
    reducersWithImmer[key] = (state, payload) =>
      typeof state === "object"
        ? produce(state, (draft: Models) => {
            const next = reducerFn(draft, payload);
            if (typeof next === "object") return next;
          })
        : reducerFn(state, payload);
  }

  return combineReducers(reducersWithImmer);
}

// rematch plugin
const immerPlugin = (): Plugin => ({
  config: {
    redux: {
      combineReducers: combineReducersWithImmer,
    },
  },
});
```

插件导出了一个 `plugin.config.redux.combineReducers` 配置，这个配置会通过[初始化方法中的 `mergeConfig()`](https://juejin.cn/post/6971404973469138957#heading-1)合入到 `rematchConfig.redux.combinReducers`，然后在[生成 rootReducer 的时候](https://juejin.cn/post/6971404973469138957#heading-6)使用。

这个 `combineReducersWithImmer`，本质上是在每一个 modelReducer 上加了一层，执行 modelReducer 时，传入的 state 其实是 `immer.produce` 的 `draftState`，因此我们可以直接修改它。当 state 为简单数据类型时，则跳过 `immer.produce`。

关于 [`immer.produce`](https://immerjs.github.io/immer/produce/) 和 [`combineReducers`](https://redux.js.org/api/combinereducers) 的更多使用和原理请参考官方文档，这里不再赘述。

**注意，关于 immer 的设计存在两个缺陷：**

1. 上面的 `immer.produce` 的第二个函数参数中，其实可以无需 `return next`。immer 会使用最终的 `draftState` 构造出新的 state。如果使用 `return newState`，则要注意不能修改 `draftState` 后，return 一个和 `draftState` 无关的新对象，[这样会报错](https://immerjs.github.io/immer/return/)。

2. 上面提到插件的 redux 相关配置会被合入到 rematch 的全局配置。但是如果使用两个插件，且两个插件都导出了这个配置，则后一个插件的配置不会被应用，下面是 merge 相关的部分代码：

```ts
for (const plugin of config.plugins) {
  if (plugin.config) {
    // redux
    if (plugin.config.redux) {
      config.redux.enhancers = [
        ...config.redux.enhancers,
        ...(plugin.config.redux.enhancers || []),
      ];
      config.redux.middlewares = [
        ...config.redux.middlewares,
        ...(plugin.config.redux.middlewares || []),
      ];
      // ... other redux configs
      config.redux.combineReducers =
        config.redux.combineReducers || plugin.config.redux.combineReducers;
      config.redux.createStore =
        config.redux.createStore || plugin.config.redux.createStore;
    }
  }
}
```

我一直觉得这部分设计让人困惑，因为很多 redux 相关的配置，如果不能像上面的 `enahancers` 和 `middlewares` 一样作为一个数组保存下来，而是被替换的话，这意味着我们插件中定义的这些配置也许无法生效。也是这个原因，在 rematch v2 中，我们实现了一个更细粒度的 plugin hooks，叫做 `onReducer`，并且使用它来重构了 immer，我会在后面的升级文章中详述。

## loading

接下来看看 loading 插件，其相对于 immer 会稍微复杂些。其主要使用的 plugin hook 是 `onModel`，最后导出一个 loading model，里面的 state 用于判断异步副作用（例如网络请求）是否正在执行中。

为了便于理解，我分为三个部分来讲述：首先是初始化代码，其次是 onModel 钩子，最后是两个 reducer。

### 初始化代码

先来看看初始化代码：

```ts
const cntState = {
  global: 0,
  models: {},
  effects: {},
};

export default (config: LoadingConfig = {}): Plugin => {
  validateConfig(config);

  const loadingModelName = config.name || "loading";

  const converter =
    config.asNumber === true ? (cnt: number) => cnt : (cnt: number) => cnt > 0;

  const loading: Model = {
    name: loadingModelName,
    reducers: {
      hide: createLoadingAction(converter, -1),
      show: createLoadingAction(converter, 1),
    },
    state: {
      ...cntState,
    },
  };

  cntState.global = 0;
  loading.state.global = converter(cntState.global);

  // ... return the plugin configs
};
```

我们定义了初始 state 变量 `cntState`，其结构包含三部分，global 用于判断全局是否有正在执行中的副作用，models 用于判断特定 model 中是否有正在执行中的副作用（例如 `loading.modelA`），而 effects 则用于判断特定的副作用是否正在执行（例如 `loading.modelA.effectA`）。

执行初始化时，首先会设置 loading 作为默认的 modelName，然后定义一个 converter，用于将 state 在 boolean 和 number 类型之间转换。（boolean 只能表示表示副作用是否处于执行中，而 number 可表示执行中的数量，0 则代表没有执行中的副作用。这样的设计便于我们按需配置），最后设置 `cntState.global` 为 0，并使用 converter 进行转换。

我们看到在定义的 loading model 中还有两个 reducer，他们俩我们会放在第三步讲解。

### onModel 钩子

其次是最重要的一部分，`onModel` 钩子的使用：

```ts
export default (config: LoadingConfig = {}): Plugin => {
  // ... 上述的初始化代码

  return {
    config: {
      models: {
        loading,
      },
    },
    onModel({ name }: Model) {
      // do not run dispatch on "loading" model
      if (name === loadingModelName) {
        return;
      }

      cntState.models[name] = 0;
      loading.state.models[name] = converter(cntState.models[name]);
      loading.state.effects[name] = {};
      const modelActions = this.dispatch[name];

      // map over effects within models
      Object.keys(modelActions).forEach((action: string) => {
        if (this.dispatch[name][action].isEffect !== true) {
          return;
        }

        cntState.effects[name][action] = 0;
        loading.state.effects[name][action] = converter(
          cntState.effects[name][action]
        );

        const actionType = `${name}/${action}`;

        // ignore items not in whitelist
        if (config.whitelist && !config.whitelist.includes(actionType)) {
          return;
        }

        // ignore items in blacklist
        if (config.blacklist && config.blacklist.includes(actionType)) {
          return;
        }

        // copy orig effect pointer
        const origEffect = this.dispatch[name][action];

        // create function with pre & post loading calls
        const effectWrapper = async (...props) => {
          try {
            this.dispatch.loading.show({ name, action });
            // waits for dispatch function to finish before calling "hide"
            const effectResult = await origEffect(...props);
            this.dispatch.loading.hide({ name, action });
            return effectResult;
          } catch (error) {
            this.dispatch.loading.hide({ name, action });
            throw error;
          }
        };

        effectWrapper.isEffect = true;

        // replace existing effect with new wrapper
        this.dispatch[name][action] = effectWrapper;
      });
    },
  };
};
```

还是分三步来看：

首先，由于 plugin 导出的 model 也会经过 `onModel` 钩子处理，因此先排除 loading model 自身。然后对 `state.models` 初始化，以及从 `this.dispatch` 中取得该 model 的所有 action。

> 注意：[前面提到过](https://juejin.cn/post/6992935103286640671#heading-0)，plugin hooks 函数中的上下文 `this` 绑定到了 pluginFactory 实例上，而 `dispatch` 作为 dispatch 插件的导出属性，也被[添加到了该 pluginFactory 实例](https://juejin.cn/post/6992935103286640671#heading-2)（不记得的同学可以翻看[前一篇文章](https://juejin.cn/post/6992935103286640671)）

其次，遍历所有 actions，从中只对 effect action 做处理：对 `state.effects[currentModel][effectAction]` 初始化，再通过黑白名单过滤掉无需处理的 effect。

最后，这也是最为重要的一步，实现对原有 effect action 的包装，包装后的 action 函数中，会在原始 effect 执行的前后，以及出错时调用 loading model 的 reducer，实现对原始 effect 执行时的状态管理。第三步让我们来分别看看这两个 reducer。

### 两个 reducer

loading model 实现了两个 reducer，分别是 `hide` 和 `show`：

```ts
const createLoadingAction =
  (converter, i) =>
  (state, { name, action }: any) => {
    cntState.global += i;
    cntState.models[name] += i;
    cntState.effects[name][action] += i;

    return {
      ...state,
      global: converter(cntState.global),
      models: {
        ...state.models,
        [name]: converter(cntState.models[name]),
      },
      effects: {
        ...state.effects,
        [name]: {
          ...state.effects[name],
          [action]: converter(cntState.effects[name][action]),
        },
      },
    };
  };

export default (config: LoadingConfig = {}): Plugin => {
  // ...

  const loading: Model = {
    name: loadingModelName,
    reducers: {
      hide: createLoadingAction(converter, -1),
      show: createLoadingAction(converter, 1),
    },
    state: {
      ...cntState,
    },
  };

  // ... return the plugin configs
};
```

`hide` 和 `show` 两个 reducer 接收的 payload 均为一个对象，对象有两个属性：modelName 和 effectName。通过这两个属性，可以找到特定的 effect 并更新其执行状态。例如，对于 `show`，执行中的 effect 对应的计数加 1，对于 `hide` 则减 1。执行中的 effect 包括全局 effect，model effect 以及特定的单个 effect。最后通过 converter 更新转换后的 state。

## 总结

到此为止，所有关于 rematch 的代码都已经讲解完毕。接下来还有两篇文章，第一篇我们来聊聊 rematch v1 升级到 v2 的一些设计变化，以及我们为什么这么做；第二篇我将注意力放到了 rematch v2 中 TypeScript 的支持上，这是升级到 v2 的最大使用变化，也是我在 rematch 团队的主要贡献，我会和大家聊聊我是如何打通 rematch 类型体系，以及这套体系目前残留的一些问题。敬请期待！
