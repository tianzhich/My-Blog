---
title: "Rematch Code Deep Dive: Part 4 - Third party plugins"
date: 2023-11-22T20:19:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["rematch", "redux", "source code"]
categories: ["Programming"]
series: ["rematch-code"]
series_weight: 5
featuredImage: "featured-image.webp"
---

This is a part of the Rematch Code Deep Dive Series, focusing on third-party plugins of Rematch.

<!--more-->

> Unless specified otherwise, the code version in this column is `@rematch/core: 1.4.0`.

In the [previous article](https://tianzhich.com/en/rematch-series-plugin-factory-and-core-plugins/), I introduced the plugin mechanism of Rematch and its two core plugins. Besides these, the Rematch team has developed several third-party plugins, such as immer, loading, select, persist, updated, etc. In this article, I will introduce two of the most commonly used plugins: immer and loading.

Before explaining them, let's first review the code structure and components of Rematch:

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

As you can see, the two core plugins mentioned above are located in the `src` directory, while the third-party plugins are in a separate `plugins` directory. In subsequent articles, I will mention that in Rematch v2, we integrated these two core plugins into `rematch/core`, so there will no longer be a `src/plugins` directory.

Below are the components of Rematch:

![rematch 组成部分](./rematch-code-structure.png)

## immer

Let's start with the immer plugin. It is inspired by [immerjs](https://github.com/immerjs/immer) and facilitates creating immutable data in reducers in a mutable way. In simple terms, without immer, the usual practice in a reducer is something like this:

```ts
function myReducer(state, action) {
  return {
    ...state,
    slice: action.payload,
  };
}
```

With immer, we can do:

```ts
function myReducer(state, action) {
  state.slice = action.payload;
}
```

Implementing the immer plugin is relatively straightforward. It involves wrapping the normal reducer to process it through immer. Let's look at the code implementation:

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

The plugin exports a `plugin.config.redux.combineReducers` configuration, which is merged into `rematchConfig.redux.combineReducers` through [the `mergeConfig()` method in the initialization process](https://tianzhich.com/en/rematch-series-rematch-core/#create-root-reducer:~:text=/src/utils/mergeConfig,%EF%BC%9A) and then used when [creating the rootReducer](https://tianzhich.com/en/rematch-series-rematch-core/#create-root-reducer).

This `combineReducersWithImmer` essentially adds a layer to each modelReducer. The state passed into the modelReducer is actually `immer.produce`'s `draftState`, so we can directly modify it. If the state is a simple data type, `immer.produce` is bypassed.

Regarding [`immer.produce`](https://immerjs.github.io/immer/produce/) and [`combineReducers`](https://redux.js.org/api/combinereducers), for more usage and principles, please refer to the official documentation. I won't repeat them here.

**However, there are two flaws in the design of immer:**

1. In the second function parameter of `immer.produce`, it's not necessary to `return next`. Immer uses the final `draftState` to construct a new state. If using `return newState`, be careful not to modify `draftState` and then return a new object unrelated to `draftState`, as [this will result in an error](https://immerjs.github.io/immer/return/).

2. The redux-related configurations mentioned in plugins are merged into Rematch's global configuration. However, if two plugins are used and both export this configuration, the latter plugin's configuration will not be applied. Below is the relevant part of the merge code:

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

I always find this part of the design confusing. Many redux-related configurations, if they cannot be saved as an array like the `enhancers` and `middlewares` above and are replaced instead, means that the configurations defined in our plugins may not be effective. For this reason, in Rematch v2, we implemented a more granular plugin hook called `onReducer`, and used it to refactor immer. I will detail this in a later article.

## loading

Next, let's look at the loading plugin, which is slightly more complex than immer. Its main usage is the `onModel` plugin hook, and it ultimately exports a loading model. The state within this model is used to determine whether asynchronous side effects (e.g., network requests) are in progress.

To make it easier to understand, I'll explain in three parts: firstly, the initialization code; secondly, the `onModel` hook; and finally, the two reducers.

### Initialization

Let's start with the initialization code:

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

We defined an initial state variable `cntState`, which consists of three parts: `global` to judge whether there are side effects in progress globally, `models` to judge whether there are side effects in a specific model (e.g., `loading.modelA`), and `effects` to judge whether a specific side effect is in progress (e.g., `loading.modelA.effectA`).

When initializing, the default modelName is set to `loading`, then a converter is defined to convert state between boolean and number types. (Boolean can only indicate whether the side effect is in progress, while number can indicate the number of side effects in progress, with 0 representing none. This design facilitates configuration as needed.) Finally, `cntState.global` is set to 0 and converted using the converter.

In the defined loading model, there are also two reducers, which we will explain in the third step.

### onModel hook

Next is the most important part, the usage of the `onModel` hook:

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

Let's look at it in three steps:

Firstly, since the model exported by the plugin also goes through the `onModel` hook, we first exclude the loading model itself. Then we initialize `state.models` and retrieve all actions of the model from `this.dispatch`.

> Note: [As mentioned before](https://tianzhich.com/en/rematch-series-plugin-factory-and-core-plugins/#plugin-factory:~:text=it%20binds%20some,of%20the%20pluginFactory.), the context `this` in the plugin hooks function is bound to the pluginFactory instance, and `dispatch`, as an exported attribute of the dispatch plugin, has also [been added to this pluginFactory instance](https://tianzhich.com/en/rematch-series-plugin-factory-and-core-plugins/#dispatch-plugin) (if you don't remember, please refer to the [previous article](https://tianzhich.com/en/rematch-series-plugin-factory-and-core-plugins)).

Secondly, we traverse all actions, processing only effect actions: initializing `state.effects[currentModel][effectAction]`, and then filtering out unnecessary effects through a whitelist or blacklist.

Finally, and most importantly, we implement the wrapping of the original effect action. The wrapped action function, before and after the execution of the original effect, as well as in case of an error, calls the reducer of the loading model to manage the state during the execution of the original effect. In the third step, let's look at these two reducers separately.

### Two reducers

The loading model implements two reducers, `hide` and `show`:

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

Both `hide` and `show` reducers receive a payload as an object with two properties: modelName and effectName. Through these two properties, a specific effect can be found and its execution status updated. For example, for `show`, the counter of the effect in progress is increased by 1, and for `hide`, it is decreased by 1. The effects in progress include global effects, model effects, and specific individual effects. Finally, the state is updated and converted through the converter.

## Summary

In conclusion, that's all about the Rematch code. In the next two articles, the first one will discuss some design changes from Rematch v1 to v2 and why we made them; the second will focus on TypeScript support in Rematch v2, which is the biggest usage change in the upgrade to v2 and is also my main contribution to the Rematch team. I will discuss how I navigated the Rematch type system and some of the current issues with this system. Stay tuned!
