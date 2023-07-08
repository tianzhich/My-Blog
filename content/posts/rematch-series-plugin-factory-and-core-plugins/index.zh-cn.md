---
title: "【Rematch 源码系列】三、Plugin factory 和 core plugins"
date: 2021-08-05T21:22:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["rematch", "redux", "源代码"]
categories: ["编程"]
series: ["rematch-code"]
series_weight: 4
featuredImage: "featured-image.webp"
---

Rematch 源码解读系列的第 3️⃣ 篇，关于 Rematch 的插件系统以及核心插件。

<!--more-->

> 如无特殊说明，本专栏文章的代码版本均为 @rematch/core: 1.4.0

[上篇](../rematch-series-rematch-core/)介绍了 rematch core 的相关代码，且其中忽视了 plugin 这一部分。这部分也是 rematch 的亮点所在，这一篇文章中我将详细介绍。

除此之外，我还会介绍 rematch 的两个 core plugins，为什么叫 core plugins，因为它们必须使用，才能让 rematch 发挥完整的功能。在下一篇文章中，我还会介绍几个第三方的 plugins，由开发者自由选择用还是不用。

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

![rematch 组成部分](./rematch-code-structure.png)

## Plugin Factory

首先是 Plugin 工厂函数，参数为 Rematch Store 初始化的参数，返回值为工厂对象，其中的关键方法属性为`create`：

```ts
export default (config: R.Config) => ({
  // ...
  create(plugin: R.Plugin): R.Plugin {
    // ... do some validations

    if (plugin.onInit) {
      plugin.onInit.call(this);
    }

    const result: R.Plugin | any = {};

    if (plugin.exposed) {
      for (const key of Object.keys(plugin.exposed)) {
        this[key] =
          typeof plugin.exposed[key] === "function"
            ? plugin.exposed[key].bind(this) // bind functions to plugin class
            : Object.create(plugin.exposed[key]); // add exposed to plugin class
      }
    }
    for (const method of ["onModel", "middleware", "onStoreCreated"]) {
      if (plugin[method]) {
        result[method] = plugin[method].bind(this);
      }
    }
    return result;
  },
});
```

主要是将 plugin 一些函数属性执行上下文的 `this` 绑定到 pluginFactory 上。如果 plugin 包含 `exposed` 属性，则将它们添加到 pluginFactory 上，用于 plugins 的共享。（在 Rematch v2 源码的升级改造中，误解了 `exposed` 属性的作用，造成了一些模棱两可的行为，后面的文章中我会提到）。最后返回一个包含了 plugin 钩子的对象。

pluginFactory 会在[前面文章](https://juejin.cn/post/6971404973469138957)提到的 Rematch 类中被调用：

```ts
export default class Rematch {
  protected config: R.Config;
  protected models: R.Model[];
  private plugins: R.Plugin[] = [];
  private pluginFactory: R.PluginFactory;

  constructor(config: R.Config) {
    this.config = config;
    this.pluginFactory = pluginFactory(config);
    for (const plugin of corePlugins.concat(this.config.plugins)) {
      this.plugins.push(this.pluginFactory.create(plugin));
    }
    // preStore: middleware, model hooks
    this.forEachPlugin("middleware", (middleware) => {
      this.config.redux.middlewares.push(middleware);
    });
  }
  public forEachPlugin(method: string, fn: (content: any) => void) {
    for (const plugin of this.plugins) {
      if (plugin[method]) {
        fn(plugin[method]);
      }
    }
  }

  // ...
}
```

在构造函数中，创建 pluginFactory 作为 Rematch 类的私有属性，然后依次将创建后的 plugin 放入一个数组。最后将 plugin 的 middleware 钩子配置到 redux 的 middleware 中。

除了 middleware 钩子，还需要依次执行 `onModel` 和 `onStoreCreated` 钩子：

```ts
export default class Rematch {
  // ...

  public addModel(model: R.Model) {
    // ...

    // run plugin model subscriptions
    this.forEachPlugin("onModel", (onModel) => onModel(model));
  }

  public init() {
    // collect all models
    this.models = this.getModels(this.config.models);
    for (const model of this.models) {
      this.addModel(model);
    }

    // ...

    this.forEachPlugin("onStoreCreated", (onStoreCreated) => {
      const returned = onStoreCreated(rematchStore);
      // if onStoreCreated returns an object value
      // merge its returned value onto the store
      if (returned) {
        Object.keys(returned || {}).forEach((key) => {
          rematchStore[key] = returned[key];
        });
      }
    });

    return rematchStore;
  }
  // ...
}
```

`onModel` 执行于遍历并添加 model 时，`onStoreCreated` 则当 store 创建完成并返回之前执行。前者通常用于读取、添加或修改 model 的配置，而后者则用于在 store 上添加新属性，如果有返回值且返回值是对象，则会将其上的属性都添加到 store 中。下面我们来看看两个具体的 plugin 以及这些钩子的应用。

## Core Plugins

Rematch v1 的设计中，有两个核心 plugin，它们在 Rematch 类的构造函数中必被引用，可以看到下面的代码片段：

```ts
export default class Rematch {
  // ...

  constructor(config: R.Config) {
    // ...

    for (const plugin of corePlugins.concat(this.config.plugins)) {
      this.plugins.push(this.pluginFactory.create(plugin));
    }

    // ...
  }

  // ...
}
```

这两个核心 plugin 分别是 dispatch 和 effects。dispatch plugin 用于增强 redux store 的 dispatch，使其支持链式调用，例如 `dispatch.modelName.reducerName`，这是 rematch 的特色之一。而 effects plugin 则用于支持异步操作等副作用，并实现通过 `dispatch.modelName.effectName` 调用。

### Dispatch plugin

先来看看 dispatch 的全部代码，然后我再将其拆解为两部分讲解：

```ts
const dispatchPlugin: R.Plugin = {
  exposed: {
    // required as a placeholder for store.dispatch
    storeDispatch(action: R.Action, state: any) {
      console.warn("Warning: store not yet loaded");
    },

    storeGetState() {
      console.warn("Warning: store not yet loaded");
    },

    /**
     * dispatch
     *
     * both a function (dispatch) and an object (dispatch[modelName][actionName])
     * @param action R.Action
     */
    dispatch(action: R.Action) {
      return this.storeDispatch(action);
    },

    /**
     * createDispatcher
     *
     * genereates an action creator for a given model & reducer
     * @param modelName string
     * @param reducerName string
     */
    createDispatcher(modelName: string, reducerName: string) {
      return async (payload?: any, meta?: any): Promise<any> => {
        const action: R.Action = { type: `${modelName}/${reducerName}` };
        if (typeof payload !== "undefined") {
          action.payload = payload;
        }
        if (typeof meta !== "undefined") {
          action.meta = meta;
        }
        return this.dispatch(action);
      };
    },
  },

  // access store.dispatch after store is created
  onStoreCreated(store: any) {
    this.storeDispatch = store.dispatch;
    this.storeGetState = store.getState;
    return { dispatch: this.dispatch };
  },

  // generate action creators for all model.reducers
  onModel(model: R.Model) {
    this.dispatch[model.name] = {};
    if (!model.reducers) {
      return;
    }
    for (const reducerName of Object.keys(model.reducers)) {
      this.validate([
        [
          !!reducerName.match(/\/.+\//),
          `Invalid reducer name (${model.name}/${reducerName})`,
        ],
        [
          typeof model.reducers[reducerName] !== "function",
          `Invalid reducer (${model.name}/${reducerName}). Must be a function`,
        ],
      ]);
      this.dispatch[model.name][reducerName] = this.createDispatcher.apply(
        this,
        [model.name, reducerName]
      );
    }
  },
};
```

#### onModel hook

前面提到，`onModel` 在 `onStoreCreated` 之前执行，因此先看看 `onModel`：

```ts
const dispatchPlugin: R.Plugin = {
  exposed: {
    // ...

    storeDispatch(action: R.Action, state: any) {
      console.warn("Warning: store not yet loaded");
    },
    dispatch(action: R.Action) {
      return this.storeDispatch(action);
    },
    createDispatcher(modelName: string, reducerName: string) {
      return async (payload?: any, meta?: any): Promise<any> => {
        const action: R.Action = { type: `${modelName}/${reducerName}` };
        if (typeof payload !== "undefined") {
          action.payload = payload;
        }
        if (typeof meta !== "undefined") {
          action.meta = meta;
        }
        return this.dispatch(action);
      };
    },
  },

  // ...

  // generate action creators for all model.reducers
  onModel(model: R.Model) {
    this.dispatch[model.name] = {};
    if (!model.reducers) {
      return;
    }
    for (const reducerName of Object.keys(model.reducers)) {
      // ... some validations

      this.dispatch[model.name][reducerName] = this.createDispatcher.apply(
        this,
        [model.name, reducerName]
      );
    }
  },
};
```

在 `onModel` 钩子中，会为每个 model 在 `this.dispatch`（上面提到，`exposed` 里的属性都会被添加到 pluginFacotry，且钩子函数上下文中的 `this` 会被绑定到这个 pluginFactory，因此 `this.dispatch` 即为上面 `exposed` 中的 `dispatch` 函数）上创建一个空对象，属性名为 model 名，然后遍历 reducer，对于每一个 reducer，以 reducer 名字作为属性名，往先前的空对象上添加一个 actionCreator。这样一来，便支持了 reducer 的链式调用。

`createDispatcher` 函数用于生成 actionCreator，里面会使用 `${model 名}/${reducer 名}` 作为 action type，最后拼装调用时传入的 payload 和 meta，使用 `this.dispatch` 调用（`this.dispatch`同时也是一个函数）。

action 派发以后，会进入到正确的 reducer 执行，关于 reducer 构造的代码在前面的[创建 Model reducers](https://juejin.cn/post/6971404973469138957#heading-5)已经讲过，这里再回顾下相关代码：

```ts
this.createModelReducer = (model: R.Model) => {
  const modelBaseReducer = model.baseReducer;
  const modelReducers = {};
  for (const modelReducer of Object.keys(model.reducers || {})) {
    const action = isListener(modelReducer)
      ? modelReducer
      : `${model.name}/${modelReducer}`;
    modelReducers[action] = model.reducers[modelReducer];
  }
  const combinedReducer = (state: any = model.state, action: R.Action) => {
    // handle effects
    if (typeof modelReducers[action.type] === "function") {
      return modelReducers[action.type](state, action.payload, action.meta);
    }
    return state;
  };

  this.reducers[model.name] = !modelBaseReducer
    ? combinedReducer
    : (state: any, action: R.Action) =>
        combinedReducer(modelBaseReducer(state, action), action);
};
```

可以看到，在 `combinedReducer` 中，会通过 `action.type` 为 action 分配正确的 reducer，而这里的 `action.type` 也由 `${model 名}/${reducer 名}` 组合而成，从而完成匹配。

#### onStoreCreated hook

再来看看最后的 `onStoreCreated` 钩子：

```ts
const dispatchPlugin: R.Plugin = {
  exposed: {
    // required as a placeholder for store.dispatch
    storeDispatch(action: R.Action, state: any) {
      console.warn("Warning: store not yet loaded");
    },

    storeGetState() {
      console.warn("Warning: store not yet loaded");
    },

    /**
     * dispatch
     *
     * both a function (dispatch) and an object (dispatch[modelName][actionName])
     * @param action R.Action
     */
    dispatch(action: R.Action) {
      return this.storeDispatch(action);
    },
  },

  // access store.dispatch after store is created
  onStoreCreated(store: any) {
    this.storeDispatch = store.dispatch;
    this.storeGetState = store.getState;
    return { dispatch: this.dispatch };
  },
};
```

由于 rematch store 是对 redux store 的增强，依赖于 redux store。因此，在 redux store 创建完成以前，是不可以访问 `storeDispatch`，`storeGetState` 和 `dispatch` 的。而创建完以后，首先是需要覆盖掉 `storeDispatch` 和 `storeGetState`，然后返回一个增强的 `dispatch`，覆盖掉 redux store 原先的 `dispatch`。

### Effect plugin

effect plugin 用于支持副作用：

```ts
const effectsPlugin: R.Plugin = {
  exposed: {
    // expose effects for access from dispatch plugin
    effects: {},
  },

  // add effects to dispatch so that dispatch[modelName][effectName] calls an effect
  onModel(model: R.Model): void {
    if (!model.effects) {
      return;
    }

    const effects =
      typeof model.effects === "function"
        ? model.effects(this.dispatch)
        : model.effects;

    for (const effectName of Object.keys(effects)) {
      // ... some validations

      this.effects[`${model.name}/${effectName}`] = effects[effectName].bind(
        this.dispatch[model.name]
      );
      // add effect to dispatch
      // is assuming dispatch is available already... that the dispatch plugin is in there
      this.dispatch[model.name][effectName] = this.createDispatcher.apply(
        this,
        [model.name, effectName]
      );
      // tag effects so they can be differentiated from normal actions
      this.dispatch[model.name][effectName].isEffect = true;
    }
  },

  // process async/await actions
  middleware(store) {
    return (next) => async (action: R.Action) => {
      // async/await acts as promise middleware
      if (action.type in this.effects) {
        await next(action);
        return this.effects[action.type](
          action.payload,
          store.getState(),
          action.meta
        );
      }
      return next(action);
    };
  },
};
```

`onModel` 钩子中做的事和 dispatch plugin 中对 reducer 的处理差不多，但也有几点不同：

1. model 配置里的 effect 参数支持函数形式，调用时，参数传入的是增强型的 `dispatch` 函数对象，返回值则是真正的 effects 对象。这样一来在 effect 中可以通过 `dispatch` 调用所有 model 的 effect 和 reducer。
2. model 中单个 effect 函数上下文的 `this` 绑定到了 `this.dispatch[model.name]`，也就是当前 model 的 `dispatch`。因此在内部可以使用 `this` 来调用当前 model 下的 reducer 和 effect。
3. 对于 effect 增加了一个标识 `isEffect` 为 `true`，用于和常规的 reducer 区分开来。

最后是 effect plugin 的核心部分，其相当于是自己实现了一个 redux 异步中间件。前面提到，effect action 也可以通过 `rematchStore.dispatch` 派发，当经过该异步中间件后，首先会判断其是否在 `this.effects` 中，如果是，首先将这个 action 往下处理，接着执行对应的 effect，传入的三个参数分别是 payload，global state 以及 meta（由于 meta 几乎很少用，而最常见的是 payload，因此采用了这样的顺序）；如果不是，则直接往下处理。

> 注意：往下处理代表着被后面的 middleware 处理（如果有），最后会来到 reducer。这里有一个容易造成困惑的地方，因为按这样的设计，一旦 model 和 effect 同名，会导致 reducer 的执行顺序在 effect 之前。两者都会执行，这也引发了 [Rematch v2 中 TS 设计部分的难处](https://github.com/rematch/rematch/pull/913)（后面文章中我会详细说明）。

## 总结

详细说了 Rematch 的插件机制，以及其核心的两个插件后，大家应该对 Rematch 巧妙的设计拍手称快了吧。在接下来的一篇文章中，我会继续介绍几个第三方的插件，我们可以选择使用，为开发提效。敬请期待！
