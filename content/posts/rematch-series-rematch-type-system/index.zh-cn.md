---
title: "【Rematch 源码系列】六、Rematch type system"
date: 2021-09-08T12:12:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["rematch", "redux", "源代码", "TypeScript"]
categories: ["编程"]
series: ["rematch-code"]
series_weight: 7
featuredImage: "featured-image.webp"
---

Rematch 源码解读系列的第 6️⃣ 篇，也是最后一篇，关于 Rematch 的 TS 类型系统。

<!--more-->

系列的最后一篇，让我们来聊聊 Rematch 背后的类型系统，这是我在 Rematch 团队的主要贡献，重构它的时候遇到了不少问题，有一些得到了解决，有一些权衡之后采取了”独特“的设计，有一些是因为 TS 的语言限制，还有一些目前我未能解决。在这篇文章中，我会把上面的问题抛出来和大家一起探讨。

> 由于相关代码较多，下面不一定全贴出来，[点此查看全部代码](https://github.com/rematch/rematch/blob/main/packages/core/src/types.ts)

## 核心类型

### Model

Rematch 中有一个很重要的概念叫做 Model，在深入 Rematch 类型系统之前，我们需要先了解它，下面是其定义：

```ts
export interface Model<
  TModels extends Models<TModels>,
  TState = any,
  TBaseState = TState
> {
  name?: string;
  state: TState;
  reducers?: ModelReducers<TState>;
  baseReducer?: ReduxReducer<TBaseState>;
  effects?: ModelEffects<TModels> | ModelEffectsCreator<TModels>;
}

export interface Models<TModels extends Models<TModels>> {
  [key: string]: Model<TModels>;
}
```

我们主要关注的是 `state`，`reducers` 和 `effects` 三个。`state` 比较简单，来看看后两个。

### ModelReducers

```ts
export type ModelReducers<TState = any> = {
  [key: string]: Reducer<TState>;
};

export type Reducer<TState = any> = (
  state: TState,
  payload?: any,
  meta?: any
) => TState;
```

`ModelReducers` 中要注意的是 `state` 类型信息的传递，因为 `Reducer` 函数的第一个参数为 `state`。

### ModelEffects 和 ModelEffectsCreator

`effects` 支持两种类型，分别是纯对象定义 `ModelEffects` 和 函数 `ModelEffectsCreator`，我们先看第一个：

**ModelEffects**

```ts
export interface ModelEffects<TModels extends Models<TModels>> {
  [key: string]: ModelEffect<TModels>;
}

export type ModelEffect<TModels extends Models<TModels>> = (
  this: ModelEffectThisTyped,
  payload: any,
  rootState: RematchRootState<TModels>,
  meta: any
) => any;

export type ModelEffectThisTyped = {
  [key: string]: (payload?: any, meta?: any) => Action<any, any>;
};
```

`ModelEffects` 和 `ModelReducers` 类似，区别是前者接收的泛型为 `TModels`，其包含了所有 Model 的信息，我们需要借助 `RematchRootState` 这个类型从中提取全局的 `rootState`，作为 `ModelEffect` 的第二个参数。而后者仅仅需要对应的 model state 信息即可。我们下面会具体讲 `RematchRootState`。

此外，`ModelEffect` 还将上下文的 `this` 绑定到了 `dispatch[currentModel]`，因此可以使用 `this[reducerName | effectName]` 的形式来派发 action。不过这部分的类型推断存在问题，我会在最后的**问题汇总**部分说明。

**ModelEffectsCreator**

```ts
export type ModelEffectsCreator<TModels extends Models<TModels>> = (
  dispatch: RematchDispatch<TModels>
) => ModelEffects<TModels>;
```

除了 `ModelEffects` 这种方式，`effects` 还可以定义为一个函数，参数为 `dispatch`，返回值为 `ModelEffects`。这样一来，在 `effects` 中不仅可以使用 上下文 `this` 来派发当前 model 的 actions，还可以使用 `dispatch` 派发所有 model 的 actions。关于 `RematchDispatch`，也放在下面讲。

### RematchRootState

了解 Redux 的都知道其核心就两个部分，一个是负责派发 action 的 dispatch，另外一个则是全局 RootState。而我们在前面学习 Rematch Model 时，出现了两个类型，`RematchRootState` 和 `RematchDispatch`，这一部分我们来详细说说。

先看看 `RematchRootState` 的类型定义：

```ts
export type RematchRootState<
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels> = Record<string, never>
> = {
  [modelKey in keyof TModels]: TModels[modelKey]["state"];
} & {
  [modelKey in keyof TExtraModels]: TExtraModels[modelKey]["state"];
};
```

简单来说就是获取各个 Model 的 state，以 modelName 作为 key，合并起来。其中，model 包含了用户自定义的 model，也包含了使用的插件导出的 model（例如使用 loading 插件就导出了 loading Model）。

#### 关于泛型 TModels 和 TExtraModels

细心的同学也许发现上面的很多类型都使用了两个泛型参数，分别是 `TModels` 和 `TExtraModels`，在 Rematch 的类型系统中，这两个泛型参数会被广泛使用。其中，`TModels` 是必填的，表示用户自定义的 Model，而 `TExtraModel` 是选填的，如果用户使用了插件且插件有导出 Model，用户就需要加上这个。

最开始的时候，我为两个泛型参数都设置了默认值 `{}`，但是 `{}` [并不意味着空类型，而是任意的非空值](https://github.com/typescript-eslint/typescript-eslint/blob/master/packages/eslint-plugin/docs/rules/ban-types.md#default-options)，因此要避免使用它。我改成了 `Record<string, any>`，不过这样也是类型不安全的。直到最后，我考虑到 `TModels` 其实是必填的（用户既然使用 Rematch 就肯定需要定义 Model），所以我删除了 `TModels` 类型参数的默认值，把 `TExtraModel` 的默认值改为 `Record<string, never>`。其实使用 `Record<string, unknown>` 也是类型安全的，但 `Record<string, unknown>` 不满足 `extends Models<TModels>` 的约束（因为 `unknown` 肯定无法赋值给 `Model`），所以换成了 `never`。

### RematchDispatch, the hybrid ReduxDispatch

再来看看 `RematchDispatch`：

```ts
export type RematchDispatch<TModels extends Models<TModels>> = ReduxDispatch & {
  [modelKey in keyof TModels]: MergeExclusive<
    ExtractRematchDispatchersFromEffects<TModel["effects"], TModels>,
    ExtractRematchDispatchersFromReducers<
      TModel["state"],
      TModel["reducers"],
      TModels
    >
  >;
};

export interface ReduxDispatch<A extends Action = AnyAction> {
  <T extends A>(action: T): T;
}
```

`RematchDispatch` 是 Rematch 的核心，整个类型推断较为复杂，因此这里我做了一些简化。首先，Rematch 的 `dispatch` 是基于 Redux `dispatch` 的一个[复合类型](https://www.typescriptlang.org/docs/handbook/interfaces.html#hybrid-types)。因此使用了 `ReduxDispatch & ...`

其次，它需要从 `effects` 和 `reducers` 配置中分别提取对应的 actions。由于 reducer 的第一个参数为 model state，因此将 `TModel['state']` 信息传入。

最后，使用 [`MergeExclusive`](https://github.com/sindresorhus/type-fest/blob/main/source/merge-exclusive.d.ts) 来合并 reducerActions 和 effectActions。一开始 Rematch 直接使用 `&` 操作符合并，但由于 Rematch 内置的每一个 action 都会被附带上 `{ isEffect: boolean }` 这样的信息，因此如果 reducer 和 effect 同名，则会出现类型不兼容的问题（因为没有类型能同时兼容 `{ isEffect: true }` 和 `{ isEffect: false }`），举个简单例子：

```ts
export interface Action<TPayload = any, TMeta = any>
  extends ReduxAction<string> {
  payload?: TPayload;
  meta?: TMeta;
}

export interface ReduxAction<T = any> {
  type: T;
}

type ReducerActions = {
  increment: ((payload: number) => void) & {
    isEffect: false;
  };
};

type EffectActions = {
  increment: ((payload: number) => void) & {
    isEffect: true;
  };
};

type Actions = ReducerActions & EffectActions;

declare const actions: Actions;

// This expression is not callable.
//   Type 'never' has no call signatures.(2349)
actions.increment(1);
```

其实 effect 和 reducer 即使同名，Rematch 代码层面也是支持的，这里先简单说下类型兼容的问题，后面关于这部分我还会继续和大家讨论。

## `createModel` helper function

除了上述核心类型，在 Rematch 中我还设计了一个工具函数 `createModel`，这个函数没有什么实际作用，只用来完善类型，以此减少用户手动添加，下面是相关代码：

```ts
export const createModel: ModelCreator =
  () =>
  (mo): any => {
    const { reducers = {}, effects = {} } = mo;

    return {
      ...mo,
      reducers,
      effects,
    };
  };

export interface ModelCreator {
  <RM extends Models<RM>>(): <
    R extends ModelReducers<S>,
    BR extends ReduxReducer<BS>,
    E extends ModelEffects<RM> | ModelEffectsCreator<RM>,
    S,
    BS = S
  >(mo: {
    name?: string;
    state: S;
    reducers?: R;
    baseReducer?: BR;
    effects?: E;
  }) => {
    name?: string;
    state: S;
    reducers: R;
    baseReducer: BR;
    effects: E;
  };
}

// 使用
export const players = createModel<RootModel>()({
  state: {
    players: [],
  } as PlayersState,
  reducers: {
    SET_PLAYERS: (state, players: PlayerModel[]) => {
      return {
        ...state,
        players,
      };
    },
  },
  effects: (dispatch) => {
    const { players } = dispatch;
    return {
      async getPlayers(payload: string, rootState): Promise<any> {
        const response = await fetch(
          "https://www.balldontlie.io/api/v1/players"
        );
        const { data }: { data: PlayerModel[] } = await response.json();
        players.SET_PLAYERS(data);
      },
    };
  },
});
```

这个函数参数为空，调用后的返回值也是一个函数，此函数参数为 `Model` 对象，并返回该 `Model` 对象自身，整个保持 `Model` 的属性类型不变。主要功能如下：

- 通过定义 `state` 的类型，打通 `reducers` 中单个 reducer 第一个参数的类型，无需重复定义
- 通过传入 `RootModel` 泛型参数，自动推断出 `effects` 中第一个参数 `dispatch` 类型
- 通过传入 `RootModel` 泛型参数，自动推断出 `effects` 中单个 effect 的第二个参数 `rootState` 类型

最开始我的 `ModelCreator` 是这样的：

```ts
interface ModelCreator {
  <RM extends Models<RM>>(): <M extends Model<RM>>(mo: M) => M;
}
```

虽然简洁了很多，但是上面第一点无法满足，因为 `state` 类型并没打通。然后我改成了：

```ts
export interface ModelCreator {
  <RM extends Models<RM>>(): <M extends Model<RM, S>, S>(mo: {
    name?: M["name"];
    state: S;
    reducers?: M["reducers"];
    baseReducer?: M["baseReducer"];
    effects?: M["effects"];
  }) => M;
}

// or
export interface ModelCreator {
  <RM extends Models<RM>>(): <M extends Model<RM, S>, S>(mo: {
    name?: M["name"];
    state: S;
    reducers?: M["reducers"];
    baseReducer?: M["baseReducer"];
    effects?: M["effects"];
  }) => {
    name?: M["name"];
    state: S;
    reducers: M["reducers"];
    baseReducer: M["baseReducer"];
    effects: M["effects"];
  };
}
```

但是上面这两种都无法保证返回值类型被推断为用户定义的 `Model` 类型，第一个的返回值类型 `M` 直接推断成 `Model<RootModel, ModelState>`。第二种虽然属性罗列出来了，但是单个属性比如 `reducers` 类型就是 `ModelReducers<ModelState>` 。也就是说，具体的单个 `reducer` 例如上面例子中的 `SET_PLAYERS` 类型没有没推导出来，上下文类型丢失。

所以最后我才采取了全展开的方式，这样一来，所有功能均可实现。

虽然功能实现了，乍一看这个函数，可能有同学会觉得奇怪：为什么使用了两个函数？

其实这是因为目前 TS 还不支持[部分类型参数推断](https://github.com/microsoft/TypeScript/pull/26349)，也就是说，如果函数指定了多个泛型参数，在调用这个函数时，要么全部由用户传参，要么参数全交给 TS 自动推断。

所以我才设计了两个函数的方案，第一个函数提供给用户传递指定泛型参数，第二个函数用户则无需指定，交给 TS 自动推断。有趣的是，设计之初我并不知道什么是「部分类型参数推断」，也并不知道这种“双函数”的设计有助于解决这个问题，后来才发现这个 PR，甚至回复里面还专门有人提到了[这个设计](https://github.com/microsoft/TypeScript/pull/26349#issuecomment-702929109)。

最后，可能有人会问为什么不单独定义一个函数类型，而是使用了 `ModelCreator` 这个 interface 定义。这是为了支持不同模块下的函数重载，将函数使用 interface 表示后，便可以使用 Module Augmentation 了，后面会单独讲讲这个方案。

## Circularly Reference

前面提到 `Model` 类型的时候，大家有没有注意到一个细节：

```ts
export interface Models<TModels extends Models<TModels>> {
  [key: string]: Model<TModels>;
}
```

仔细看上面的 `Models` 泛型参数 `TModels` 的约束条件为 `Models<TModels>`。是不是有点迷惑，其实这也是我无意之中设计的，最开始的时候，我是直接使用了默认参数：

```ts
export interface Models<TModels extends Models = Record<string, any>> {
  [key: string]: Model<TModels>;
}
```

但上面毕竟使用了 `any`，作为一个严格要求自己的”体操队员“，我当然要减少 `any` 出现的情况，因此我打算将其改为 `never` 或 `unknown`。但是我很快又发现了问题：

如果 `Model` 的泛型参数 `TModels` 使用 `Record<string, never>`：那么作为 `Model` 属性之一的 `effects` 类型则为 `ModelEffectsCreator<Record<string, never>>`，又由于 `ModelEffectsCreator` 是一个函数，其参数便推断为 `RematchDispatch<Record<string, never>>`。

在我[前面的文章中](https://juejin.cn/post/6844904166536527880)提到过，函数参数兼容是逆变的，因此此处只需要判断`RematchDispatch<Record<string, never>>` 是否兼容 `RematchDispatch<TModels>`（即后者是否可以赋值给前者）。

我们继续对 `RematchDispatch` 进行解析，其中需要从 `Model` 中提取 `effects` 信息：

```ts
/**
 * Extracts a dispatcher for each effect that is defined for a model.
 */
export type ExtractRematchDispatchersFromEffectsObject<
  TEffects extends ModelEffects<TModels>,
  TModels extends Models
> = {
  [effectKey in keyof TEffects]: ExtractRematchDispatcherFromEffect<
    TEffects[effectKey],
    TModels
  >;
};
```

而由于此时的 `Model` 为 `never`，那么 `effects` 也会是 `never`，因此上面的索引 `effectKey` 也会为 `never`。而我们知道，`never` 无法兼容除了其自身的任何类型，也就是说除了它自身的任何类型都无法赋值给它，因此上面的 `RematchDispatch<TModels>` 就无法赋值给 `RematchDispatch<Record<string, never>>`。所以最终会报索引签名的兼容性错误，下面是错误栈：

```ts
// Type 'Model<Record<string, never>, any, any>' is not assignable to type 'Model<TModels, any, any>'.
//  Types of property 'effects' are incompatible.
//    Type 'ModelEffects<Record<string, never>> | ModelEffectsCreator<Record<string, never>> | undefined' is not assignable to type 'ModelEffects<TModels> | ModelEffectsCreator<TModels> | undefined'.
//      Type 'ModelEffectsCreator<Record<string, never>>' is not assignable to type 'ModelEffects<TModels> | ModelEffectsCreator<TModels> | undefined'.
//        Type 'ModelEffectsCreator<Record<string, never>>' is not assignable to type 'ModelEffectsCreator<TModels>'.
//          Types of parameters 'dispatch' and 'dispatch' are incompatible.
//            Type 'RematchDispatch<TModels>' is not assignable to type 'RematchDispatch<Record<string, never>>'.
//              Type 'RematchDispatch<TModels>' is not assignable to type 'ExtractRematchDispatchersFromModels<Record<string, never>>'.
//                Index signatures are incompatible.
//                  Type 'any' is not assignable to type 'never'.ts(2344)
```

既然 `never` 行不通，换成 `unknown` 呢？

```ts
export interface Models<TModels extends Models = Record<string, unknown>> {
  [key: string]: Model<TModels>;
}
```

显然这里使用 `unknown` 甚至无法满足 `extends Models` 的约束，因为 `unknown` 是肯定不能赋值给 `Model` 的。

最后，由于我始终认为用户既然使用 Rematch，肯定会定义 `TModels`，因此这个泛型参数其实是不需要有默认值的。所以我删除了默认值，但此时没有了默认值，`extends Models` 肯定是不行了，我就碰巧改成了 `extends Models<TModels>`，最终就变成了下面这种看起来有些奇怪的循环引用模样：

```ts
export interface Models<TModels extends Models<TModels>> {
  [key: string]: Model<TModels>;
}
```

虽然一切都正常了，但我其实并不了解这种写法，只知道它可以解决我当下的问题。说实话，如果不是写这篇文章，这个地方我可能不会深入研究，但通过写作可以倒逼我去了解这种写法。搜索的过程中，还发现了一个不错的[解释](https://stackoverflow.com/a/50277451/14251417)。

回到我的问题上来，为什么我会使用这样的方法？首要原因是我需要**创建约束**，前面提到，`Model` 需要获取所有的 models 信息，而每一个单独的 `Model` 又作为 `Models` 的一个属性，因此为了打通类型，我需要给 `Models` 也加上 `TModels` 的泛型参数，而且 `TModels` 需要满足约束，因为它肯定也是 `Models` 子集，它代表着用户自定义的全局 models 类型。所以最终 `TModels` 的约束就为 `extends Models<TModels>`。可能还是有点绕口，下面我举一个[实际例子](https://github.com/rematch/rematch/blob/main/examples/all-plugins-react-ts/src/models/index.ts)：

```ts
export interface RootModel extends Models<RootModel> {
  players: typeof players;
  cart: typeof cart;
  settings: typeof settings;
}

export interface Models<TModels extends Models<TModels>> {
  [key: string]: Model<TModels>;
}
```

把 `TModels` 换成 `RootModel` 代入，是不是好理解很多。这里我其实就是想保证 `TModels` 为 `Models` 的子集（也就是用户实际创建的 `RootModel`），而不是 `Models` 自身。如果还难以理解，可以直接查看[该回答](https://stackoverflow.com/a/50277451/14251417)，回答中的例子应该更恰当一些。

答主还提到，[TS 支持多态 `this`](https://www.typescriptlang.org/docs/handbook/advanced-types.html#polymorphic-this-types)，因此我这里的定义其实可以绕开繁琐且难以理解的“循环”，改为更优雅的方式：

```ts
export interface Models {
  [key: string]: Model<this>;
}
```

这里的 `this` 就也能表示用户自定义的 models 了。值得注意的是这与直接替换成 `Models` 不同，如果使用 `Model<Models>`，而此时的 `Models` 无法表示用户自定义的 models，它的 key 都是 string 类型，这样传入给 `Model` 是没有实际意义的。

当然，上面这种方法我还没实践，后期我会找个时间提出一个 PR，来继续重构这部分代码。

## Module Augmentation

TS 里有一个功能叫做[声明合并](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)，针对第三方的模块和全局环境，分别有 Module Augmentation 和 Global Augmentation 两种合并模式，Augmentation 有「增强，增加」的意思，表示扩展模块或全局的功能（类型层面）。

Global Augmentation 这里简单提一下，如果在模块中（文件包含 `import`, `export` 关键字），需要将额外的声明放入 `declare global {}` 下面：

```ts
// observable.ts
export class Observable<T> {
  // ... still no implementation ...
}
declare global {
  interface Array<T> {
    toObservable(): Observable<T>;
  }
}
Array.prototype.toObservable = function () {
  // ...
};
```

而如果本来就是全局脚本文件，则无需添加 `declare global {}` 块。

这里主要讲讲 Module Augmentation，Rematch 有 3 个地方使用了这种模式，用来消除类型不兼容。

### @rematch/core

首先是 `declare module '@rematch/core' {}`，在 select 和 typed-state 插件中均有使用。先来看看 select 插件：

```ts
declare module "@rematch/core" {
  // Add overloads for store to add select
  interface RematchStore<
    TModels extends Models<TModels>,
    TExtraModels extends Models<TModels>
  > extends ReduxStore<RematchRootState<TModels, TExtraModels>, Action> {
    select: RematchSelect<
      TModels,
      TExtraModels,
      RematchRootState<TModels, TExtraModels>
    >;
  }

  // add overloads for Model here.
  interface Model<TModels extends Models<TModels>, TState = any> {
    selectors?: ModelSelectorsConfig<TModels, TState>;
  }

  // add overloads for ModelCreator here.
  interface ModelCreator {
    <RM extends Models<RM>>(): <
      R extends ModelReducers<S>,
      BR extends ReduxReducer<BS>,
      E extends ModelEffects<RM> | ModelEffectsCreator<RM>,
      SE extends ModelSelectorsConfig<RM, S>,
      S,
      BS = S
    >(mo: {
      name?: string;
      state: S;
      selectors?: SE;
      reducers?: R;
      baseReducer?: BR;
      effects?: E;
    }) => {
      name?: string;
      state: S;
      selectors: SE;
      reducers: R;
      baseReducer: BR;
      effects: E;
    };
  }
}
```

select 插件在 `Model` 中增加了一个 `selectors` 属性，同时导出了一个 `select` 函数，挂在 `RematchStore` 上。

在前面讲解 `createModel` helper 时，提到了 `ModelCreator` 为什么定义成了一个 interface。由于 select 插件是支持在 model 里定义 selectors 的，所以这里可以方便地利用 Module Augmentation 进行函数重载。

typed-state 插件主要是提供给以纯 JS 使用 Rematch 的开发者，通过 `typings` 配置，方便在开发环境中发现一些错误定义的类型，它也使用了 Module Augmentation：

```ts
declare module "@rematch/core" {
  interface Model<
    TModels extends Models<TModels> = Record<string, any>,
    TState = any
  > {
    typings?: Record<string, any>;
  }

  // add overloads for ModelCreator here.
  interface ModelCreator {
    <RM extends Models<RM>>(): <
      R extends ModelReducers<S>,
      BR extends Reducer<BS>,
      E extends ModelEffects<RM> | ModelEffectsCreator<RM>,
      S,
      BS = S
    >(mo: {
      name?: string;
      state: S;
      reducers?: R;
      baseReducer?: BR;
      effects?: E;
      typings?: Record<string, any>;
    }) => {
      name?: string;
      state: S;
      typings?: Record<string, any>;
      reducers: R;
      baseReducer: BR;
      effects: E;
    };
  }
}
```

### redux

`@rematch/core` 中针对 `redux` 也使用了 Module Augmentation：

```ts
declare module "redux" {
  export interface Dispatch<A extends Action = AnyAction> {
    [modelName: string]: any;
  }
}
```

前面提到 `RematchDispatch` 时，讲到它其实是一个复合类型，由 Rematch 自己的 dispatcher 联合上 ReduxDispatch（`& ReduxDispatch`），下面是 `ReduxDispatch`：

```ts
export interface ReduxDispatch<A extends Action = AnyAction> {
  <T extends A>(action: T): T;
}
```

而增加上面的定义，并使用 `any`，可以消除很多源码中的类型报错。对于使用 Rematch 的开发者来说，models 信息都是提前定义好的，可是在源码中只能使用泛型 `TModels` 表达，这里面有不少错误，有部分我也没发现原因，只是用 `any` 来避免它们，可以说这部分也算是一个残留的问题，感兴趣的同学可以去看看源码，如果能解决，欢迎 PR。

可能有人会问为什么还要额外定义一个 `RematchDispatch`，直接使用 Module Augmentation 不也可以吗？这是因为 TS 限制同名的声明需要使用相同的泛型参数，而 `RematchDispatch` 还需要 `TModels` 信息，无法和 `ReduxDispatch` 保持一致。

### react-redux

针对 `react-redux` 的 `connect` 方法，Rematch 也做了兼容，由于 `connect` 只能识别 `ReduxDispatch`，所以需要对它进行重载，使其也可以支持 `RematchDispatch`：

```ts
declare module "react-redux" {
  interface Connect {
    <RM extends Models<RM>, State, TStateProps, TDispatchProps, TOwnProps>(
      mapStateToProps: MapStateToPropsParam<TStateProps, TOwnProps, State>,
      mapDispatchToProps: MapRematchDispatchToPropsNonObject<
        TDispatchProps,
        TOwnProps,
        RM
      >
    ): InferableComponentEnhancerWithProps<
      TStateProps & TDispatchProps,
      TOwnProps
    >;
  }

  type MapRematchDispatchToPropsNonObject<
    TDispatchProps,
    TOwnProps,
    RM extends Models<RM>
  > =
    | MapRematchDispatchToPropsFactory<TDispatchProps, TOwnProps, RM>
    | MapRematchDispatchToPropsFunction<TDispatchProps, TOwnProps, RM>;

  type MapRematchDispatchToPropsFactory<
    TDispatchProps,
    TOwnProps,
    RM extends Models<RM>
  > = (
    dispatch: RematchDispatch<RM>,
    ownProps: TOwnProps
  ) => MapRematchDispatchToPropsFunction<TDispatchProps, TOwnProps, RM>;

  type MapRematchDispatchToPropsFunction<
    TDispatchProps,
    TOwnProps,
    RM extends Models<RM>
  > = (dispatch: RematchDispatch<RM>, ownProps: TOwnProps) => TDispatchProps;
}
```

通过重载，我们引入了 `RM` 泛型，它就是 `TModels` ，这样一来，`dispatch` 参数便可以兼容了，从而 `connect` 也兼容了。

这里多提一句，在 Redux 的 [StyleGuide](https://redux.js.org/style-guide/style-guide#use-the-react-redux-hooks-api) 中，更建议使用 hooks，也就是 `useSelector` 和 `useDispatch` 来替代 `connect`，对于 Rematch 也是一样的，Redux 官方也认为 `connect` 的类型定义实在过于复杂，不易使用，过多的函数重载，可选参数，还需要合并 props 等等，感兴趣可以点击上面的链接去看看。

> 注意：前面提到同名的声明需要使用相同的泛型参数，不过如果这个声明用于函数重载，里面的重载函数的泛型参数是可以不同的。

## 问题汇总

最后的部分，我挑了几个典型的问题，有一些已经得到了解决，也有部分暂时未能解决。这些问题我都搜索了大量资料，其中还发现了一些 TS 的设计限制（design limitation），它们都十分有趣，抛出来和大家探讨。

### 【已解决】dispatcher inference

原始的 PR 请[点我查看](https://github.com/rematch/rematch/pull/901)。

前面提到过 `RematchDispatch`，这个其实是 Rematch 的一个核心，而且类型的推导也比较复杂。比如我们在 model 中定义的 reducer 有三个参数，分别是 `modelState`，`payload` 和 `meta`，但是 dispatch 调用时只要传递 `payload` 和 `meta` 即可。在 effect 中也是三个参数，分别是 `payload`，`rootState` 和 `meta`，在调用时传递 `payload` 和 `meta`。

提取参数并生成新的函数定义这一过程看似简单，实则也踩了不少坑。

先看看 reducer 的提取：

```ts
export type ExtractRematchDispatcherFromReducer<TState, TReducer> =
  TReducer extends () => any
    ? RematchDispatcher
    : TReducer extends (state: TState, ...args: infer TRest) => TState
    ? TRest extends []
      ? RematchDispatcher
      : RematchDispatcher<TRest[0], TRest[1]>
    : never;

export type RematchDispatcher<TPayload = void, TMeta = void> = [
  TPayload,
  TMeta
] extends [void, void]
  ? (() => Action<void, void>) & { isEffect: false }
  : [TMeta] extends [void]
  ? undefined extends TPayload
    ? ((payload?: TPayload) => Action<TPayload, void>) & {
        isEffect: false;
      }
    : ((payload: TPayload) => Action<TPayload, void>) & {
        isEffect: false;
      }
  : [undefined, undefined] extends [TPayload, TMeta]
  ? ((payload?: TPayload, meta?: TMeta) => Action<TPayload, TMeta>) & {
      isEffect: false;
    }
  : undefined extends TMeta
  ? ((payload: TPayload, meta?: TMeta) => Action<TPayload, TMeta>) & {
      isEffect: false;
    }
  : ((payload: TPayload, meta: TMeta) => Action<TPayload, TMeta>) & {
      isEffect: false;
    };
```

注意这里的 `RematchDispatcher` 使用了 `void` 作为泛型的默认参数。直到写这篇文章，我才发现一个 bug。因为 `void` 除了其自身，只能赋值给 `any` 和 `unknown`，但是反过来的行为却很怪异：

```ts
// What types are compatible with `void`?
type case1 = [any] extends [void] ? 1 : 0; // 1
type case2 = any extends void ? 1 : 0; // 0 | 1
type case3 = unknown extends void ? 1 : 0; // 0
type case4 = [unknown] extends [void] ? 1 : 0; // 0

// What types are `void` compatible with?
type case5 = void extends any ? 1 : 0; // 1
type case6 = void extends unknown ? 1 : 0; // 1
```

由于上面的 case1 成立，当 reducer 的第二个参数 `payload` 被用户自己定义成 `any` 时，生成的 dispatch 函数会没有参数，这显然不对。

其实，`void` 类型的兼容性比 `never` 好，`never` 只能兼容其自身，但对于 `any` 则和 `void` 表现一样，不过使用 `[]` 以后则不一样：

```ts
// What types are compatible with `never`?
type case1 = [any] extends [never] ? 1 : 0; // 0
type case2 = any extends never ? 1 : 0; // 0 | 1
```

利用这一点，我后面会提一个 PR 来修复这个问题。

回到正题，推导的思路如下：

- 如果用户没有定义参数，或者只使用了 state -> 参数为空的 dispatch

- 否则提取第一个参数 `payload` 和 第二个参数 `meta`：
  - 如果未定义 `meta`，即 `[TMeta] extends [void]`：
    - `payload` 可选，即 `undefined extends TPayload` -> `payload` 为可选参数的 dispatch
    - 否则 -> `payload` 为必选参数的 dispatch
  - 否则，提取 `TMeta`，并判断：
    - 如果 `meta` 和 `payload` 均可选，即 `[undefined, undefined] extends [TPayload, TMeta]` -> 两个参数均可选的 dispatch
    - 否则，`payload` 必选，并判断 `TMeta`：
      - `TMeta` 可选，即 `undefined extends TMeta` -> `payload` 必选，`meta` 可选的 dispatch
      - 否则 -> 两个参数均为必选的 dispatch

可以观察到，上面我们做了一个优化，比如即使用户定义了 `(state, payload: number | undefined)`，在生成 dispatch 函数时，对应的 `payload` 也会是可选的，这在逻辑上是合理的（这里值得更多讨论，也有部分人认为这种形式的参数应该是必选，哪怕传一个 `undefined`），但正常的定义还是 `payload?: number`。

reducer 的推导相对简单，因为 `state` 作为第一个参数，用户定义的参数都在它后面。但 effect 则复杂一些，主要体现在两个方面：

1. effect 可以为一个对象，也就是上面提到的 `ModelEffects`，还可以为一个函数 `ModelEffectsCreator`
2. effect 的 `rootState` 参数位于第二个，`payload` 位于第一个，而 `meta` 在最后

关于第二点，有人可能会问为什么不统一。因为最初设计的时候，是考虑了实际使用情况的，在 reducer 中，一般都需要拿到当前 modelState，所以把它放在了第一个参数，而在 effect 中，大多时候是只需要 `payload` 的，因此把 `rootState` 挪到了中间，而最不常使用的 `meta` 则都放到最后。

那我们来看看 effect 的推导：

```ts
export type ExtractRematchDispatchersFromEffects<
  TEffects extends Model<TModels>["effects"],
  TModels extends Models<TModels>
> = TEffects extends (...args: any[]) => infer R
  ? R extends ModelEffects<TModels>
    ? ExtractRematchDispatchersFromEffectsObject<R, TModels>
    : never
  : TEffects extends ModelEffects<TModels>
  ? ExtractRematchDispatchersFromEffectsObject<TEffects, TModels>
  : never;

export type ExtractRematchDispatchersFromEffectsObject<
  TEffects extends ModelEffects<TModels>,
  TModels extends Models<TModels>
> = {
  [effectKey in keyof TEffects]: TEffect extends (
    ...args: infer TRest
  ) => infer TReturn
    ? TRest[1] extends undefined
      ? EffectRematchDispatcher<TReturn, TRest[0]>
      : RematchRootState<TModels> extends TRest[1]
      ? EffectRematchDispatcher<TReturn, TRest[0], TRest[2]>
      : never
    : never;
};
```

首先是使用 `extends (...args: any[])` 来判断 effect 是对象还是函数，如果是函数则需要提取返回值，否则直接使用。重点来看看第二步：这里我使用了一个巧妙的方式，那就是先判断 `rootState` 参数，如果它为 `undefined` 说明用户没有定义该参数，则只需要考虑 `TRest[0]` 也就是 `payload` 即可。其次核对一下 `rootState` 的类型，这里为什么使用 `RematchRootState<TModels> extends TRest[1]` 而不是反过来呢？因为 `rootState` 这里作为第二个参数，存在一种情况：**用户可以将第一个参数 `payload` 定义为可选，而 TS 不允许必选参数跟在可选后面**，所以需要把 `rootState` 也定义为可选，在这种情况下，由于参数逆变，就必须使用上面的顺序。更多信息可以参考这个[讨论](https://github.com/rematch/rematch/discussions/924)。关于 TS 为什么有这样的限制，这也有个不错的[回答](https://stackoverflow.com/a/46960442/14251417)。

如果 `rootState` 合法，则分别提取 `TRest[0]` 和 `TRest[2]`，并附带上返回值信息 `TReturn`，传递给 `EffectRematchDispatcher`。之后要做的事就和 reducer 一样了，唯一不同的是多了一个 `TReturn`，effect 允许用户自定义返回值，而 reducer 返回的必须是一个 `ReduxAction`。

注意：使用 `infer TRest` 来提取参数，还有一个比较好的地方，就是如果参数未定义，传入 `EffectRematchDispatcher` 时，如果泛型使用了默认值，则会使用默认值。这和直接把参数定义为 `undefined` 不一样。

### 【已解决】type guard

原始的 issue [请点我查看](https://github.com/microsoft/TypeScript/issues/40429)。

我们先来看一个[代码片段](https://github.com/microsoft/TypeScript/issues/10530#issuecomment-603915317)：

```ts
const obj: { prop: string | null } = { prop: "hello" };

if (typeof obj.prop === "string") obj.prop.length; //OK

if (typeof obj["prop"] === "string") obj["prop"].length; //OK

const key = "prop" as const;
if (typeof obj[key] === "string") obj[key].length; //error - object is possibly 'null'
```

上面的 error 确实是 TS 的一个问题，且现在仍然存在。一个解决方案是：

```ts
const key = "prop" as const;
const configWorks = config[key];
if (typeof configWorks !== "boolean") {
  configWorks.prop = "test"; // ok
}
```

但是我当时简化出来的[例子](https://www.typescriptlang.org/play?ts=4.0.2#code/JYOwLgpgTgZghgYwgAgAoBsCuBzUAJAewIGtkBvAKGWuQJAGUwCoIBhFuSAEwH4AuZAAoADgEYBAZzBRQ2AJTIAvAD5kANwLAuAbio06AWQJcI6fkOEAmSdNkKV6zTooBfChRMJ0cFsgR0pZGEsXBAJAQwcfCJiAG0AXV0KGEwQBDBgOmQUtIAeQhJkCAAPSBAuCWRiCABPAhg0EOiSZUEAWwgwAAtjAQLiABpskAERAQA5OnHMdG8AI3QIXMjQ-tjquoaV5uJ45XtVDS05ASOucj1qGGYhfzCwIKaQWgbgqLCFShpvvwCHnsKike71iHW6xkSlx+wAagjANWEEHqyABpAAhIogQAiVImGCgCBcLGfKE-GgwECCVFyXRkskAenpyAAeqzmaS6fSAFQcskAQSg2EwHXAL2Q8MRyAA5NsQGt+vEpchgJUQAQHnAJBJgNgQHAFigmEEfHAwdAxRKUFLBCJxMgpDIQPIlIcnAoAD5CETWe22J0HRzHKUAOl5dIAKgirTaxDZHc6HGcPV6rHG7C7A1xk7iIPiQISlSrkGqNVqdXqDeKCOKo9KY3aHenE27kJ6Yz7G-6M0mQ2G6chI5KpTm8wXlar1chNdrdfrFlWa0P62mu83jq2Ux2-QnXUHQ-2DwPazKnvKYorx8XJ9Py3PDdXLXXU774wGzr3D4fB9HbSud5nk3bP83xbT0RwJLhCwnUsZwrecjUfa1n07f9333T8v2PcD80gy8SynMtZ0rBDj29YDuycD9Py5ekwzcfsYSEakLgwikqRiGk+xoRkWTZLjqG5fjkAFIURQeZFENlM8SAvIt8JvIj4OrYQTTNKALVI38XybXcsw3IDtNXXS5CojDv2lKSYliBUoKvGDb2Ih9SOQ7cQK0UyDxouiw0YwQ0TRZivk-NjqVpT8eLZdkMMEjCROFCBRQk49LJIazz1s+TCLg+9jSgU1OnNJKly0lC3L0tsyMM1C3Q8zChxSuIbLw68srvBdEMq0qKPc9DDy8-t6JoNw3CAA)更复杂，而且使用了上面的解决方案也没用。虽然这个问题后来在 TS v4.3.x 中修复了，但当时我是怎么做的呢？

由于传递的参数我使用了 `NonNullable`，既然 `if` 语句判断后无法构造出一个 `NonNullable` 类型，那么便自己手动构造：

```ts
// working method 1
if (typeof hook !== "undefined") {
  fn(hook!);
}

// working method 2
const hoook = hook ?? undefined;
if (typeof hoook !== "undefined") {
  fn(hoook);
}
```

当时发现使用 [Non-null assertion 操作符](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-0.html#non-null-assertion-operator)和 [Nullish coalescing 操作符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing_operator)都可以达到效果。前者是 TS 的语法，感觉是直接给你构造了一个 `NonNullable` 类型，而后者是 JS 语法，这样操作以后，`hoook` 的值变成了 `undefined | NonNullable<hook>`，然后再经过 `if`，则把 `undefined` 过滤了。

特别是最后一个方法，感觉就很神奇，本来自身是可能为 `undefined` 的（我称之为隐式的），现在这个“隐式 `undefined`”被转化成了显式的 `undefined`，就可以被过滤了。当然，这只是我的一个描述，具体原因也没明白，而且后面这个问题被 TS 修复了，就没有再去深究。

### 【已解决】distributive conditional types

原始的 issue 请[点我查看](https://github.com/microsoft/TypeScript/issues/42072)。

这个问题其实是和上面提到的 dispatcher inference 有关，而且应该是在我重写这部分之前提出来的。那个时候我并不知道什么是 [distributive conditional type](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html#distributive-conditional-types)，且前面也提到过，`Dispatcher` 的类型推导实在过于复杂（前面是我后来优化的，之前的判断更复杂），看着大段的条件分支，也不知道错误是从哪个分支开始出现的。

记得当时我用的都是很蠢的方法，就是人工把条件拆开，一步一步判断，才终于发现了蹊跷，但由于就是不知道 Distributive conditional types 这个概念，搜索也费了很大劲。最后终于发现还有这个概念，突然豁然开朗。下面是一个简单的例子：

```ts
type NakedExample<T> = T extends void ? "true" : "false";

type OptionalNumber = number | undefined;

// 'true' | 'false'
type Foo = NakedExample<OptionalNumber>;

// 'false'
type Bar = OptionalNumber extends void ? "true" : "false";
```

官方说明如下：

> Conditional types in which the checked type is a naked type parameter are called distributive conditional types. Distributive conditional types are automatically distributed over union types during instantiation. For example, an instantiation of T extends U ? X : Y with the type argument A | B | C for T is resolved as (A extends U ? X : Y) | (B extends U ? X : Y) | (C extends U ? X : Y).

我还咨询了一下为什么要这么设计，[RyanCavanaugh](https://github.com/RyanCavanaugh)的回答如下：

> Distributivity is usually what you want in those scenarios, so it's the default when the checked type is a naked type parameter. This was in favor of making special syntax for distributivity/non-distributivity since, with this behavior, you just generally don't have to think about it.

大概意思就是这确实是一个很好的设计，而且这个设计是很符合我们的常规思考的（这也是为什么我之前没发现什么蹊跷），而且这种分发泛型的设计可以用来开发出很多工具类型：

```ts
type Diff<T, U> = T extends U ? never : T; // Remove types from T that are assignable to U
type Filter<T, U> = T extends U ? T : never; // Remove types from T that are not assignable to U

type NonNullable<T> = Diff<T, null | undefined>; // Remove null and undefined from T

type T34 = NonNullable<string | number | undefined>; // string | number
```

我们熟知的 `Pick`, `Exclude` 等等类型都是基于这个设计。

---

讲了这么多已解决的问题，其实未解决的问题也有很多，部分是我目前的能力所限，部分也是 TS 的限制。如果大家读完以后可以帮助参与 Rematch 的贡献，真是感激不尽！接下来一起看看那些未解决的问题吧。

---

### 【未解决】circularly reference

原始的 issue 请[点我查看](https://github.com/microsoft/TypeScript/issues/40279)。

前面提到过 **Circularly Reference**，主要针对 `Models` 这个类型。而这里还有一种循环引用的[情况](https://www.typescriptlang.org/play?ts=4.4.2#code/C4TwDgpgBAogHsATgQwMbADwCUD2PhQQIQB2AJgM5RYSo6JkYVICWJA5gDRTIkgB8-KAF4oAbwBQUaVADaAawggobKIpA4AZtTzAAugC4d+BUr0SAvhImhIsTZtrAKGACpDRACjBH4SNJjuAJQiQgBuOCxk1kRg9AR0JMxQqIgQyMAQRti6hMTkVDR0DEysHNy8AvyeIcJCGFIyMHmZBfaO6C64+PycjdLVALY4RpIy44QOThQAggD8vn0TMhBTnQBCC7D9UBa1QmPLkx3OM4s746snFJu+OxYiUDWhT2BBRpUvYNZsmYiaaGg3QIh2kmjwRlsEC0UHBOEs1kSyThj1S6UyOR6NU8oOO0zOTzILAoYAyqAAFvtxBcZAB6WlQADu9EUZCZLGA5JwAFcCGQIMxENz0Gx2FAiSSyeSadIkTgADYQAB08pw7E8EtJwApSrhQQA3Pclis1s51kYcbC8LsqbjxvTYcgWIq2YyOeTxQKkMLgKKZSkcEkFcrVeq9YbxhY+nsgA)：

```ts
type Extract<Root extends Record<string, any>> = {
  [key in keyof Root]: Root[key];
};

type Effects<T> = (p: Extract<T>) => void;

export const create: <Root extends Record<string, any>>() => <
  E extends Effects<Root>
>(mo: {
  effectsA?: E;
  effectsB?: E;
}) => {
  effectsA: E;
  effectsB?: E;
} =
  () =>
  (p): any =>
    p;

interface Root {
  foo: typeof foo;
}

const foo = create<Root>()({
  effectsA: (dispatch) => {
    // worked without destructing dispatch
    console.log(dispatch.foo);
  },
  effectsB: ({ foo }) => {
    // failed with destructing
    console.log(foo);
  },
});
```

可以看到，在 `effectsB` 这种写法里面，TS 是会报循环引用错误的，但是这在代码层面其实是 OK 的，因为 `effectsB` 和 `effectsA` 的写法几乎一致，唯一不同的是 `effectsA` 函数中始终通过 `dispatch` 来访问 `foo`，而 `effectsB` 则是先解构，这可能会造成 `effectsB` 中的 `foo` 始终为 `undefined`，因为 `foo` 可能是后面才被添加到 `dispatch` 中的，对于这种问题，使用 `effectsA` 这种方式可以避免。Rematch 就曾出现过这样的[问题](https://github.com/rematch/rematch/issues/811)。

虽然上面的问题解决了（方案可以查看问题详情链接），但是 TS 层面还会认为存在循环引用：

- `Root['foo']` 的类型为 `typeof foo`
  - 要知道 `foo` 的类型，需要知道 `effectsB` 的参数类型
    - `effectsB` 的参数类型为 `Extract<Root>`
      - 解构的 `foo` 类型为 `Extract<Root>['foo']` 也就是 `Root['foo']`
        - 回到第一个

这个问题一直没有得到解决，而且还存在几个类似的问题，有一些也比较奇怪，感兴趣可以参考下面的一些评论：

- https://github.com/microsoft/TypeScript/issues/35546#issuecomment-611485331
- https://github.com/microsoft/TypeScript/issues/35546#issuecomment-645036905
- https://github.com/microsoft/TypeScript/issues/35546#issuecomment-668287404

### 【未解决】partial arg inference

原始的 issue 请[点我查看](https://github.com/microsoft/TypeScript/issues/41668)。

这个 issue 其实涉及到两个问题，其中的一个和 `createModel` 工具函数那个部分说的一样，涉及到[部分类型参数推断](https://github.com/microsoft/TypeScript/pull/26349)，而另外一个，则是我想得简单了，比如下面这个[简化的例子](https://www.typescriptlang.org/play?ts=4.4.2#code/KYDwDg9gTgLgBAE2AYwDYEMrDjAnmbAWQiVQCVgEBXZYKAZzgF44BvAWACgBIAbQGtguAFxx6MKAEsAdgHMAuqIrVaULgF8uXUJFiIUGLDnzZlNOszgAKLtzDpcqCOgSj003FwCUzAHxwANwhJBC1OJDRMbGQIaXE4ZCx0GGBiUlEAHjI4UBTpBEY04HJKcwZfKwBbCFFWOCwVOnolOHUfJn8gkLCIw2jY+MTgZNSSYoB1SRgACwgqGDNVAGUJSWQYSQHMsgrq2vrS1Wa4bLa-QODQzi4Aehu4e0dnBAfMdErGGTgAA3pgGCWEEq-2mMlk3wS7jg0gg8AARtgZAAzOiUODoRjSKiVBFqTgxOLwJEQCCWIYjIqoKwcThwA6NBi1Lh0ul3azAmYkHx-AFAkFgqyPJwuAD8bg8XlEXSuLLE-0BHNBckFDmFLxYAEYfDTZZpaa0NF4wmyhc9XlB3p9pD8eQr+XIIcgoQi4MjUS8MdDsbiuAT4nDMGSkilKZNOfNFnQVlJ1ps4tTmfSyscdSy2VUQVy5bzFQLTaLRFicXRJRduvq6ba+TM86qzZrtYm6Xrm4bjfd8y97BaPq7rb95dWleDIdaXW6GujMd6LAB3KbTHLgVBrKbGAjo6QwmDJOO+gbw9AALyDwxDYypqYayaZFbg6cVWarueVnbFXuLUFL0qb2btNdfOsXELGcoEsLU2F-FsDU4NouCAA)：

```ts
export declare type ModelReducers = {
  [key: string]: Reducer;
};

export declare type Reducer = (payload: any) => void;

declare const createModel: <R extends ModelReducers>(mo: {
  reducers: R;
}) => void;

declare const createModelWithoutReducerStrictions: <R>(mo: {
  reducers: R;
}) => void;

// payload params in `setSomething` can not be infered as number
const foo = createModel({
  reducers: {
    // (method) setSomething(payload?: any): void
    setSomething(payload = 1) {},
  },
});

// payload params in `setSomething` can be infered as number
const bar = createModelWithoutReducerStrictions({
  reducers: {
    // (method) setSomething(payload?: number): void
    setSomething(payload = 1) {},
  },
});

// payload params in `setSomething` can be infered as number with explicit type annotation
const baz = createModel({
  reducers: {
    // (method) setSomething(payload?: number): void
    setSomething(payload: number = 1) {},
  },
});
```

在上面的例子的 `foo` 中，我本以为在 `setSomething` 中使用默认参数 `payload = 1` 就可以将 `payload` 推断为 `number`。后来想明白了，由于我们对 `createModel` 中的 `reducers` 做了类型约束，不管是使用参数默认赋值，还是显式声明参数类型，都需要确保这个类型和约束的类型是兼容的。但默认赋值并不能改变推断的类型。

其实，在 Rematch 代码中，`Reducer` 的类型是这样的：

```ts
export declare type Reducer<TState = any> = (
  state: TState,
  payload?: any
) => TState;
```

前面提到过，由于无法做部分类型推断（也就是这里的 `TState` 使用用户定义的泛型，而 `payload` 使用推断），所以我们把 `payload` 定义为了一个可选的 `any` 类型，这样一来，由于任何类型都兼容 `any`，所以用户在实际定义 `payload` 时可以缩小它的类型。

> 注意：上面的 `payload` 可选和必选效果是一样的，因为 `undefined | any` 等于 `any`。

### 【未解决】同名 reducer 和 effect 的类型设计

前面提到，在 Rematch 中 `reducer` 可以和 `effect` 使用相同命名。而且调用时，`reducer` 会先执行，其次是 `effect`。我也不太清楚最初为什么这么设计，而且这样的行为对用户来说是隐藏的。

除此之外，这也对类型的设计造成了很大挑战，甚至说根本无法做到。下面是 effect 中间件的代码片段：

```ts
function createEffectsMiddleware<
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels>
>(bag: RematchBag<TModels, TExtraModels>): Middleware {
  return (store) =>
    (next) =>
    (action: Action): any => {
      if (action.type in bag.effects) {
        // first run reducer action if exists
        next(action);

        // then run the effect and return its result
        return (bag.effects as any)[action.type](
          action.payload,
          store.getState(),
          action.meta
        );
      }

      return next(action);
    };
}
```

上述代码会先判断 `action.type` 是否存在于 `effects` 中，如果有，则先调用 `reducer`（`next(action)` 表示调用下一个中间件，最后为执行 reducer）。

那么，类型层面，该怎么考虑？使用 union 提示两种类型？或者是考虑只提供 reducer 类型？我觉得两个方案都不太合理，虽然现在 [Sergio](https://github.com/semoal) 采用的是方案 2，这个方案在上面讲 RematchDispatch 的时候提到过，这里再来[回顾](https://www.typescriptlang.org/play?#code/JYWwDg9gTgLgBAbzgWQKZQOaoKIA8DGANgK4DOwAbqnAL5wBmUEIcA5DAJ5ioC09qpGKwBQwztzgAlVABNi+dAEF8MYBAB2pOAF5EwuAbjB1+KKhCp1MAFxwAFHbABDDoQhOZt9cRAAjdACUOgB8cBQQwDJBAGR6hvFGpNj0-Cq29E6EpKgA3PqGNHnxMqim5pY29o4ubh62glDGGEHaoeGRMXEJBsBJKaWVGVm5+QaFo0YmZhZWAMIQYBy2Ds6u7p5w3n7ocAA+cA1NLW0RUXCxCBPxvcmpg5nZRQV542Jc1LcDyqoaWrqX1ym5Ssy2qazqmx8-igewOMEa6maIUh2ygnQB3USnzScHhxBG8XGxVK0wqoNWtQ2W2hxxRNPOXW6N36OLxBOeE2MZRmMHmi3JNXWXihgWR1NFFyuhmZd1sbKeYxeeTeEm+ak0ACFUPRoNRdNI5AooGrfgzsTATZo8uJqJbSIp6DAdro0JgcAQSOQqAAeA3yJQqdWkAA0cHNduCypKRCcZjg+F+8Ccgd+Wp1ZlsdrTury0cIseoCc0SZTmgdTqgmdL9sd6GVwgA9A24AAVAAWvTgqFwYDMpHIGkSmwg8HwmXzvkIqAAdI3mwYW+82OpUFQoKw4G2nFp1BB4+ODsAMOonDBiH3p3YAEwAZgALABOALCZM-TXa3XTrmkqx2ACMATKk29i9gs6CcEE37ApUYKUsKqK0uKML7O0MgvtW5boF+QI8v+z5znAiiYD4FRwBA9C4kurBIRuna7km-ZHiek7UDAe7OFATgWBWZEUTay6rugrCXred4AKzPq+QaYVA07Rj+MB4UBzZERgJFWLxlESKwhyIrRO4jnA27kMeTgsbi7GxlxqA8eRWnUNRIrriJ94Sehb41hW2HchUfIcHYrB-qw+FAA)下：

```ts
import { MergeExclusive } from "type-fest";

type ReducerActions = {
  increment: ((payload: number) => void) & {
    isEffect: false;
  };
  decrement: ((payload: string) => void) & {
    isEffect: false;
  };
  incrementCopy: ((payload: number | string) => void) & {
    isEffect: false;
  };
};

type EffectActions = {
  increment: ((payload: number | string) => number) & {
    isEffect: true;
  };
  decrement: ((payload: number) => number) & {
    isEffect: true;
  };
  incrementCopy: ((payload: number) => number) & {
    isEffect: true;
  };
};

type ActionsBefore = ReducerActions & EffectActions;
type ActionsAfter = MergeExclusive<ReducerActions, EffectActions>;

declare const actionsBefore: ActionsBefore;
declare const actionsAfter: ActionsAfter;

// This expression is not callable.
//   Type 'never' has no call signatures.(2349)
actionsBefore.increment(1);

// (property) increment: (payload: number) => number | void
actionsAfter.increment(1);

// Argument of type 'number' is not assignable to parameter of type 'never'.(2345)
actionsAfter.decrement(1);

// Argument of type 'string' is not assignable to parameter of type 'number'.(2345)
actionsAfter.incrementCopy("1");
```

由于 `isEffect: boolean` 这个设置的影响，同名 reducer 和 effect 会导致 `never` 类型的出现，函数无法调用。所以我们换成了 [`MergeExclusive`](https://github.com/sindresorhus/type-fest/blob/main/source/merge-exclusive.d.ts) 这个工具类型来做这件事，这个工具类型也比较好理解。比如有两个类型 `A` 和 `B`，其最终是使用联合来替代交叉，但是在联合前它对 `A` 和 `B` 分别做了两个处理：

- 对于 `A`，将其与 `B` 不同的属性全设置为 `?: never`（由于可选也相当于 `undefined`，这里实际上就是 `undefined | never`，也就是 `undefined`）
- 再与 `B` 相交

同样，对于 `B` 再做一遍，然后将它们联合。

回到前面，这个方法的优缺点分别是什么呢？优点就是当 reducer 和 effect 同名，但是 `payload` 完全不同时，由于使用了联合类型，会导致 `payload` 为 `never` 从而无法调用，参考上面代码中的 `actionsAfter.decrement(1)`，为什么说是优点？因为这恰好符合 Rematch 的设计，因为经过 reducer 先处理后的 action，还会继续传到 effect，如果这俩的 `payload` 类型完全不一致，那么显然可能导致错误。

那么缺点呢？缺点也同样是因为联合类型，看上面代码中的 `actionsAfter.increment(1)`，reducer 的 `payload` 类型是 effect 的 `payload` 类型的子集，这是符合预期的，比如上面会提示 `payload: number`，这样代码也能正常执行。但如果反过来，TS 仍然可以保证代码的成功运行，可由于 reducer 先执行，本意是提示 reducer 的类型，这里却提示了 effect，见上面代码中的 `actionsAfter.incrementCopy('1')`。

说实话，同名的 reducer 和 effect 确实很奇怪，我们应该避免这种情况。

### 【未解决】`this` types

原始的 issue 请[点我查看](https://github.com/rematch/rematch/issues/870)。

在本专栏的[第三篇文章](https://juejin.cn/post/6992935103286640671#heading-5)，讲 Rematch 的核心插件时，提到了 effect 函数的上下文 `this` 被绑定到了 `dispatch[modelName]`。这样做可以方便地在 effect 中使用 `this` 来派发当前 model 的所有 actions。但是，这也给 TS 层面的类型兼容带来了挑战。目前的 effect 类型定义如下：

```ts
export type ModelEffect<TModels extends Models<TModels>> = (
  this: ModelEffectThisTyped,
  payload: Action["payload"],
  rootState: RematchRootState<TModels>,
  meta: Action["meta"]
) => any;
```

而如果把它改为：

```ts
export type ModelEffect<TModel, TModels extends Models<TModels>> = (
  this: ModelDispatcher<TModel>,
  payload: Action["payload"],
  rootState: RematchRootState<TModels>,
  meta: Action["meta"]
) => any;
```

这样的话，几乎所有类型都要增加 `TModel` 泛型参数，而且也会造成 `Model` 变成下面这样：

```ts
export interface Model<
  TModel extends Model<TModel, TModels>,
  TModels extends Models<TModel, TModels>,
  TState = any,
  TBaseState = TState
> {
  name?: string;
  state: TState;
  reducers?: ModelReducers<TState>;
  baseReducer?: ReduxReducer<TBaseState>;
  effects?:
    | ModelEffects<TModel, TModels>
    | ModelEffectsCreator<TModel, TModels>;
}
```

这样一来，`Model` 也循环引用自身了，前面的 **Circularly Reference** 部分中的 `Models` 也是一样，且我提到也许可以使用 `this` 来表示类型中的自身。如果可行，我觉得对于 `Model` 也可以使用这种方法。目前我还没有太多时间，但我会持续关注这两个问题。

### 【未解决】select plugin types

最后一个未解决的问题，便是如何完善 select 插件的类型，这个插件原作者的代码写得比较复杂，甚至我看了 [reselect](https://github.com/reduxjs/reselect) 源码，都觉得比这个简单，我没办法完全理解，因此该插件的类型定义也就是稍微完善了一下，很多地方其实没走通。如果有感兴趣的同学，可以了解一下，顺便能修复就更不错了。

## 总结

其实在重构 Rematch 类型系统的初期，我的”体操“水平是相当不足的，所以你会看到我的很多设计都是碰巧、偶然实现的，只是发现这样可行，感到很神奇。但是通过这篇文章，我溯源了很多所谓”奇怪“的设计，并发现了 TS 更多有趣的地方。希望大家在学习的时候，也一定要知其然并知其所以然，保持热爱！

本篇专栏到此就结束了，希望大家通过读完所有的文章，能更深入地了解 Rematch，从而高效地开发，玩转状态管理。
