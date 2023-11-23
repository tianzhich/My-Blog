---
title: "Rematch Code Deep Dive: Part 6 - The Rematch type system"
date: 2023-11-23T11:41:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["rematch", "redux", "source code", "TypeScript"]
categories: ["Programming"]
series: ["rematch-code"]
series_weight: 7
featuredImage: "featured-image.webp"
---

This is the final article of the Rematch Code Deep Dive Series, focusing on Rematch's type system.

<!--more-->

This final part of the series discusses the type system behind Rematch, which was my main contribution to the Rematch team. While refactoring it, I encountered numerous problems: some were resolved, some required "unique" design decisions due to trade-offs, some were limitations of the TS language, and others remain unsolved. In this article, I will discuss these issues for further exploration.

> Due to the extensive related code, not all of it is included below. [Click here to view all the code](https://github.com/rematch/rematch/blob/main/packages/core/src/types.ts).

## Core Types

### Model

In Rematch, a key concept is the Model. Before diving deep into the Rematch type system, we need to understand this concept. Here is its definition:

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

We are mainly concerned with `state`, `reducers`, and `effects`. `State` is straightforward, so let's look at the latter two.

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

In `ModelReducers`, it's important to note that the first parameter of the `Reducer` function is `state`.

### ModelEffects and ModelEffectsCreator

`Effects` support two types: plain object-defined `ModelEffects` and function `ModelEffectsCreator`. Let's start with the first one:

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

`ModelEffects` and `ModelReducers` are similar, but the former receives the generic type `TModels`, which contains information about all Models. We use the `RematchRootState` type to extract the global `rootState` as the second parameter of `ModelEffect`. The latter only requires the corresponding model state information. We will discuss `RematchRootState` below.

Moreover, `ModelEffect` binds the context's `this` to `dispatch[currentModel]`, so actions can be dispatched using `this[reducerName | effectName]`. However, there are issues with type inference here, which I will address in the final **problem summary** section.

**ModelEffectsCreator**

```ts
export type ModelEffectsCreator<TModels extends Models<TModels>> = (
  dispatch: RematchDispatch<TModels>
) => ModelEffects<TModels>;
```

Besides the `ModelEffects` method, `effects` can also be defined as a function taking `dispatch` as a parameter and returning `ModelEffects`. This allows to dispatch actions of the current model using the context `this` in `effects`, as well as dispatching actions of all models using `dispatch`. We will discuss `RematchDispatch` later.

### RematchRootState

Those familiar with Redux know it has two core parts: the action-dispatching dispatch and the global RootState. When studying Rematch Model earlier, we encountered two types, `RematchRootState` and `RematchDispatch`. Let's discuss them in detail.

First, let's look at the type definition of `RematchRootState`:

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

Simply put, it obtains each Model's state, combining them with `modelName` as the key. Models include both user-defined models and those exported by used plugins (e.g., using the loading plugin exports the loading Model).

#### About Generics type `TModels` and `TExtraModels`

Attentive readers may have noticed that many types above use two generic parameters, `TModels` and `TExtraModels`, extensively used in Rematch's type system. `TModels` is mandatory, representing user-defined Models, while `TExtraModel` is optional, used if the user utilizes plugins exporting Models.

Initially, I set default values for both generic parameters as `{}`, [but `{}` does not mean an empty type; it represents any non-empty value](https://github.com/typescript-eslint/typescript-eslint/blob/master/packages/eslint-plugin/docs/rules/ban-types.md#default-options), so I changed it to `Record<string, any>`. However, this was also type-unsafe. 

Eventually, considering `TModels` is mandatory (since users of Rematch will definitely define Models), I removed the default value for TModels and changed the default value for `TExtraModel` to `Record<string, never>`. Using `Record<string, unknown>` is also type-safe, but it does not satisfy the `extends Models<TModels>` constraint (since `unknown` cannot be assigned to `Model`), so I switched to `never`.

### RematchDispatch, the hybrid ReduxDispatch

Next, let's look at `RematchDispatch`:

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

Firstly, `RematchDispatch` is the core of Rematch, with complex type inference, so I've simplified it here. First, Rematch's `dispatch` is a [hybrid type](https://www.typescriptlang.org/docs/handbook/interfaces.html#hybrid-types) based on Redux `dispatch`. Hence the use of `ReduxDispatch & ....`.

Secondly, it needs to extract corresponding actions from `effects` and `reducers`. Since the reducer's first parameter is model state, `TModel['state']` information is passed in.

Lastly, reducerActions and effectActions are combined using [`MergeExclusive`](https://github.com/sindresorhus/type-fest/blob/main/source/merge-exclusive.d.ts). Initially, Rematch directly used the `&` operator to combine, but as each action in Rematch carries information like `{ isEffect: boolean }`, if a reducer and effect share the same name, a type incompatibility issue arises (since no type can simultaneously be compatible with `{ isEffect: true }` and `{ isEffect: false }`). Here's a simple example:

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

In truth, even if an effect and reducer have the same name, Rematch's code supports it. I will simply discuss the type compatibility issue here and continue the discussion with everyone later.

## `createModel` helper function

Besides the core types, I also designed a utility function `createModel` in Rematch. This function doesn't have any practical effect and is only used to perfect types, reducing the need for users to manually add them. Here is the related code:

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

// how to use
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

The function parameter is empty, and the return value is also a function, whose parameter is a `Model` object and returns the `Model` object itself, keeping the `Model`'s attribute types unchanged. The main functions are as follows:

- By defining the type of `state`, the type of the first parameter in reducers is passed through, avoiding repetitive definitions.
- By passing in the `RootModel` generic parameter, the type of the first parameter `dispatch` in effects is automatically inferred.
- By passing in the `RootModel` generic parameter, the type of the second parameter `rootState` in individual effects is automatically inferred.

Initially, `ModelCreator` was like this:

```ts
interface ModelCreator {
  <RM extends Models<RM>>(): <M extends Model<RM>>(mo: M) => M;
}
```

Although much simpler, the above could not meet the first point as the `state` type was not connected. Then I changed it to:

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

However, the above two approaches fail to ensure that the return type is inferred as the user-defined `Model` type. In the first approach, the return type `M` is directly inferred as `Model<RootModel, ModelState>`. In the second approach, although the attributes are listed, the type of an individual attribute like `reducers` is just `ModelReducers<ModelState>`. This means that the specific individual reducer type, like `SET_PLAYERS` in the above example, is not deduced, and the contextual type is lost.

That's why I eventually adopted a fully expanded approach, which made all functionalities achievable.

Although this method works, at first glance, one might wonder why two functions were used.

The reason is that TypeScript currently does not support [partial type parameter inference](https://github.com/microsoft/TypeScript/pull/26349). This means that if a function has multiple generic parameters, when calling this function, either all parameters must be provided by the user, or they are all inferred by TS automatically.

Therefore, I devised a two-function approach: the first function allows the user to pass specific generic parameters, while the second function does not require the user to specify anything, leaving it to TS for automatic inference. Interestingly, I was not aware of "partial type parameter inference" at the beginning of the design and did not know that this "double function" design could help solve this problem. I only discovered this PR later, and someone even specifically mentioned [this design](https://github.com/microsoft/TypeScript/pull/26349#issuecomment-702929109) in the replies.

Finally, one might ask why I didn't just define a single function type, but instead used the `ModelCreator` interface definition. This was to support function overloading in different modules. By representing the function with an interface, Module Augmentation can be utilized, which I will discuss separately later.

## Circularly Reference

When I previously mentioned the `Model` type, did you notice a detail?

```ts
export interface Models<TModels extends Models<TModels>> {
  [key: string]: Model<TModels>;
}
```

Take a close look at the generic parameter `TModels` of `Models`, which is constrained to `Models<TModels>`. It might be a bit confusing, but this was an unintended design on my part. Initially, I directly used a default parameter:

```ts
export interface Models<TModels extends Models = Record<string, any>> {
  [key: string]: Model<TModels>;
}
```

But the above approach indeed uses `any`, and as someone who strictly disciplines themselves as a "TS Gymnast," I of course want to reduce the occurrence of `any`. Therefore, I planned to change it to `never` or `unknown`. However, I quickly encountered a problem:

If the generic parameter `TModels` of `Model` uses `Record<string, never>`, then the type of `effects`, one of the properties of `Model`, would be `ModelEffectsCreator<Record<string, never>>`. Since `ModelEffectsCreator` is a function, its parameter would be inferred as `RematchDispatch<Record<string, never>>`.

As I mentioned in [a previous article](https://juejin.cn/post/6844904166536527880), function parameter compatibility is contravariant, so here we only need to determine whether `RematchDispatch<Record<string, never>>` is compatible with `RematchDispatch<TModels>` (i.e., whether the latter can be assigned to the former).

Let's continue to analyze `RematchDispatch`, where we need to extract `effects` information from `Model`:

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

But since `Model` at this time is `never`, then `effects` will also be `never`, so the index `effectKey` above will also be `never`. As we know, `never` cannot be compatible with any type other than itself, meaning any type other than itself cannot be assigned to it. Therefore, the above `RematchDispatch<TModels>` cannot be assigned to `RematchDispatch<Record<string, never>>`. So, in the end, it will report an error of compatibility with the index signature. Below is the error stack:

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

Since `never` didn't work, what about `unknown`?

```ts
export interface Models<TModels extends Models = Record<string, unknown>> {
  [key: string]: Model<TModels>;
}
```

Clearly, using `unknown` here doesn't even satisfy the constraint of `extends Models`, because `unknown` definitely cannot be assigned to `Model`.

Finally, because I always believed that if users are using Rematch, they would definitely define `TModels`, this generic parameter actually does not need a default value. So, I removed the default value, but without a default value, `extends Models` definitely wouldn't work, so I happened to change it to `extends Models<TModels>`, and it eventually became the below somewhat strange circular reference:

```ts
export interface Models<TModels extends Models<TModels>> {
  [key: string]: Model<TModels>;
}
```

Although everything now works normally, I don't actually understand this kind of writing. I only knew it could solve my current problem. To be honest, if it weren't for writing this article, I might not have studied this place deeply, but writing forced me to understand this kind of writing. In the process of searching, I even found a good [explanation](https://stackoverflow.com/a/50277451/14251417).

Returning to my problem, why did I use this method? The primary reason is that I needed to **create a constraint**. As mentioned earlier, `Model` needs to get all the models' information, and each individual `Model` acts as an attribute of `Models`. Therefore, to connect types, I needed to add the generic parameter `TModels` to `Models`, and `TModels` needs to meet the constraint, as it definitely is also a subset of `Models`, representing the user-defined global models type. So, the final constraint of `TModels` became `extends Models<TModels>`. It might still sound a bit convoluted, so let me give an [actual example](https://github.com/rematch/rematch/blob/main/examples/all-plugins-react-ts/src/models/index.ts):

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

If we substitute `TModels` with `RootModel`, it becomes much easier to understand. Here, I'm essentially trying to ensure that `TModels` is a subset of `Models` (i.e., the actual `RootModel` created by the user), not `Models` itself. If it's still hard to understand, you can directly look at [this answer](https://stackoverflow.com/a/50277451/14251417), where the examples in the answer might be more appropriate.

The author also mentioned that [TS supports polymorphic `this`](https://www.typescriptlang.org/docs/handbook/advanced-types.html#polymorphic-this-types), so my definition here could actually bypass the cumbersome and hard-to-understand "circular" and be changed to a more elegant way:

```ts
export interface Models {
  [key: string]: Model<this>;
}
```

This `this` can also represent the user-defined models. It's worth noting that this is different from directly replacing it with `Models`. If we use `Model<Models>`, and at this time `Models` cannot represent the user-defined models, its keys are all string types, making it meaningless to pass to `Model`.

Of course, I haven't practiced the above method yet. Later, I will find time to submit a PR to continue refactoring this part of the code.

## Module Augmentation

TS has a feature called [declaration merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html), which has two merging modes for third-party modules and global environments: Module Augmentation and Global Augmentation, respectively. Augmentation means "enhancement, addition," referring to expanding the functionality (at the type level) of a module or globally.

Global Augmentation is briefly mentioned here. If in a module (a file containing `import`, `export` keywords), additional declarations need to be placed under `declare global {}`:

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

If it's already a global script file, then there's no need to add the `declare global {}` block.

Here, let's mainly talk about Module Augmentation. Rematch uses this mode in 3 places to eliminate type incompatibility.

### @rematch/core

First is `declare module '@rematch/core' {}`, used in both the select and typed-state plugins. Let's first look at the select plugin:

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

The select plugin adds a `selectors` attribute to `Model` and also exports a `select` function, mounted on `RematchStore`.

When explaining the `createModel` helper earlier, I mentioned why `ModelCreator` is defined as an interface. Since the select plugin supports defining selectors in the model, it can conveniently utilize Module Augmentation for function overloading.

The typed-state plugin is mainly for developers using Rematch purely in JS. Through the `typings` configuration, it helps to identify some erroneously defined types in the development environment. It also uses Module Augmentation:

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

`@rematch/core` also uses Module Augmentation for `redux`:

```ts
declare module "redux" {
  export interface Dispatch<A extends Action = AnyAction> {
    [modelName: string]: any;
  }
}
```

When previously mentioning `RematchDispatch`, I talked about it being a composite type, combined with Rematch's own dispatcher and `ReduxDispatch` (`& ReduxDispatch`). Below is `ReduxDispatch`:

```ts
export interface ReduxDispatch<A extends Action = AnyAction> {
  <T extends A>(action: T): T;
}
```

Adding the above definition and using `any` can eliminate many type errors in the source code. For developers using Rematch, models information is predefined, but in the source code, it can only be expressed using the generic `TModels`. There are several errors in this area, some of which I couldn't find the reason for and just used `any` to avoid them. This part could be considered a lingering issue, and interested developers are welcome to check out the source code and submit PRs if they can resolve it.

You might wonder why we need to define a separate `RematchDispatch` instead of just using Module Augmentation. This is because TS requires declarations with the same name to use the same generic parameters, and `RematchDispatch` needs `TModels` information, which cannot be consistent with `ReduxDispatch`.

### react-redux

Rematch also made compatibility adjustments for the `connect` method in `react-redux`. Since `connect` can only recognize `ReduxDispatch`, it needs to be overloaded to support `RematchDispatch`:

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

By overloading, we introduce the generic `RM`, which is `TModels`, so that the `dispatch` parameter can be compatible, making `connect` compatible as well.

It's worth mentioning that, according to Redux's [StyleGuide](https://redux.js.org/style-guide/style-guide#use-the-react-redux-hooks-api), it is recommended to use hooks, i.e., `useSelector` and `useDispatch`, instead of `connect`. This is also applicable to Rematch. The Redux team also believes that `connect`'s type definitions are overly complex and difficult to use, with too many function overloads, optional parameters, and the need to merge props, etc. Interested readers can check the link for more details.

> While same-named declarations require the same generic parameters, if the declaration is used for function overloading, the overloaded functions' generic parameters can be different.

## Problem Summary

Finally, I picked a few typical issues, some of which have been resolved and some are still unresolved. I searched a lot of material for these issues and even found some design limitations of TS, all of which are quite interesting and worth discussing.

### 【Resolved】dispatcher inference

The original PR can be viewed [here](https://github.com/rematch/rematch/pull/901).

I previously mentioned `RematchDispatch`, which is actually a core part of Rematch and has complex type inference. For example, a reducer defined in a model has three parameters: `modelState`, `payload`, and `meta`, but when calling dispatch, only `payload` and `meta` need to be passed. In an effect, there are also three parameters: `payload`, `rootState`, and `meta`, with `payload` and `meta` being passed during the call.

Extracting parameters and generating new function definitions seems simple but has its challenges.

Let's look at the extraction of reducers:

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

Note that `RematchDispatcher` uses `void` as the default generic parameter. It was only while writing this article that I discovered a bug. Because `void`, apart from itself, can only be assigned to `any` and `unknown`, but the reverse behavior is quite strange:

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

Due to the validity of case1 above, when the reducer's second parameter `payload` is defined by the user as `any`, the generated dispatch function will have no parameters, which is clearly incorrect.

In fact, the compatibility of `void` is better than `never`, with `never` only being compatible with itself, but `any` behaves the same as `void`. However, using `[]` later changes this:

```ts
// What types are compatible with `never`?
type case1 = [any] extends [never] ? 1 : 0; // 0
type case2 = any extends never ? 1 : 0; // 0 | 1
```

I will use this point to submit a PR later to fix this issue.

Returning to the main topic, the inference logic is as follows:

- If the user hasn't defined parameters, or only used state -> the dispatch parameter is empty.
- Otherwise, extract the first parameter `payload` and the second parameter `meta`:
  - If `meta` is not defined, i.e., `[TMeta] extends [void]`:
    - `payload` is optional, i.e., `undefined extends TPayload` -> dispatch with an optional `payload`
    - Otherwise -> dispatch with a mandatory `payload`
  - Otherwise, extract `TMeta` and determine:
    - If both `meta` and `payload` are optional, i.e., `[undefined, undefined] extends [TPayload, TMeta]` -> dispatch with both parameters optional
    - Otherwise, `payload` is mandatory, and determine `TMeta`:
      - `TMeta` is optional, i.e., `undefined extends TMeta` -> dispatch with mandatory `payload` and optional `meta`
      - Otherwise -> dispatch with both parameters mandatory

It can be observed that we made an optimization above. For instance, even if the user defines `(state, payload: number | undefined)`, the corresponding `payload` in the generated dispatch function will be optional, which is logically reasonable (this deserves more discussion, and some people believe that such parameters should be mandatory, even if passing an `undefined`). But the normal definition is still `payload?: number`.

Reducer inference is relatively simple because `state` is the first parameter, and the user-defined parameters are all behind it. But effect inference is more complex, mainly reflected in two aspects:

1. An effect can be an object, that is, the `ModelEffects` mentioned above, or a function `ModelEffectsCreator`.
2. In the effect, the `rootState` parameter is second, `payload` is first, and `meta` is last.

Regarding the second point, one might ask why not unify it. Because the original design considered practical usage. In reducers, the current modelState is generally needed, so it is placed as the first parameter. In effects, most of the time, only `payload` is needed, so `rootState` is moved to the middle, and the least used `meta` is placed last.

Let's look at effect inference:

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

First, we use `extends (...args: any[])` to determine whether the effect is an object or a function. If it's a function, we need to extract the return value; otherwise, use it directly. The key point to look at in the second step: here, I used a clever approach, which is to first judge the `rootState` parameter. If it is `undefined`, it means the user hasn't defined this parameter, so we only need to consider `TRest[0]`, which is `payload`. Next, check the type of `rootState`. Why use `RematchRootState<TModels> extends TRest[1]` instead of the reverse? Because `rootState` here is the second parameter, there is a situation: **the user can define the first parameter `payload` as optional, and TS does not allow mandatory parameters to follow optional ones**, so `rootState` also needs to be defined as optional. In this case, due to parameter contravariance, we must use the above order. More information can be found in this [discussion](https://github.com/rematch/rematch/discussions/924). For why TS has such a restriction, there's also a good [answer](https://stackoverflow.com/a/46960442/14251417).

If `rootState` is valid, then extract `TRest[0]` and `TRest[2]`, along with the return information `TReturn`, and pass it to `EffectRematchDispatcher`. The subsequent steps are the same as for reducers, the only difference being the additional `TReturn`. Effects allow users to define their own return values, while reducers must return a `ReduxAction`.

> Using `infer TRest` to extract parameters has another advantage: if the parameter is not defined, when passed to `EffectRematchDispatcher`, if the generic uses a default value, the default value will be used. This is different from directly defining the parameter as `undefined`.

### 【Resolved】type guard

The original issue can be viewed [here](https://github.com/microsoft/TypeScript/issues/40429).

Let's first look at a [code snippet](https://github.com/microsoft/TypeScript/issues/10530#issuecomment-603915317):

```ts
const obj: { prop: string | null } = { prop: "hello" };

if (typeof obj.prop === "string") obj.prop.length; //OK

if (typeof obj["prop"] === "string") obj["prop"].length; //OK

const key = "prop" as const;
if (typeof obj[key] === "string") obj[key].length; //error - object is possibly 'null'
```

The above error is indeed a problem with TS, and it still exists now. One solution is:

```ts
const key = "prop" as const;
const configWorks = config[key];
if (typeof configWorks !== "boolean") {
  configWorks.prop = "test"; // ok
}
```

But the [example](https://www.typescriptlang.org/play?ts=4.0.2#code/JYOwLgpgTgZghgYwgAgAoBsCuBzUAJAewIGtkBvAKGWuQJAGUwCoIBhFuSAEwH4AuZAAoADgEYBAZzBRQ2AJTIAvAD5kANwLAuAbio06AWQJcI6fkOEAmSdNkKV6zTooBfChRMJ0cFsgR0pZGEsXBAJAQwcfCJiAG0AXV0KGEwQBDBgOmQUtIAeQhJkCAAPSBAuCWRiCABPAhg0EOiSZUEAWwgwAAtjAQLiABpskAERAQA5OnHMdG8AI3QIXMjQ-tjquoaV5uJ45XtVDS05ASOucj1qGGYhfzCwIKaQWgbgqLCFShpvvwCHnsKike71iHW6xkSlx+wAagjANWEEHqyABpAAhIogQAiVImGCgCBcLGfKE-GgwECCVFyXRkskAenpyAAeqzmaS6fSAFQcskAQSg2EwHXAL2Q8MRyAA5NsQGt+vEpchgJUQAQHnAJBJgNgQHAFigmEEfHAwdAxRKUFLBCJxMgpDIQPIlIcnAoAD5CETWe22J0HRzHKUAOl5dIAKgirTaxDZHc6HGcPV6rHG7C7A1xk7iIPiQISlSrkGqNVqdXqDeKCOKo9KY3aHenE27kJ6Yz7G-6M0mQ2G6chI5KpTm8wXlar1chNdrdfrFlWa0P62mu83jq2Ux2-QnXUHQ-2DwPazKnvKYorx8XJ9Py3PDdXLXXU774wGzr3D4fB9HbSud5nk3bP83xbT0RwJLhCwnUsZwrecjUfa1n07f9333T8v2PcD80gy8SynMtZ0rBDj29YDuycD9Py5ekwzcfsYSEakLgwikqRiGk+xoRkWTZLjqG5fjkAFIURQeZFENlM8SAvIt8JvIj4OrYQTTNKALVI38XybXcsw3IDtNXXS5CojDv2lKSYliBUoKvGDb2Ih9SOQ7cQK0UyDxouiw0YwQ0TRZivk-NjqVpT8eLZdkMMEjCROFCBRQk49LJIazz1s+TCLg+9jSgU1OnNJKly0lC3L0tsyMM1C3Q8zChxSuIbLw68srvBdEMq0qKPc9DDy8-t6JoNw3CAA) I simplified at the time was more complex, and using the above solution didn't work. Although this issue was later fixed in TS v4.3.x, what did I do at the time?

Since I used `NonNullable` for the passed parameters, if the `if` statement couldn't construct a `NonNullable` type, then I would construct it manually:

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

I found that using both the [Non-null assertion operator](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-0.html#non-null-assertion-operator) and the [Nullish coalescing operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing_operator) could achieve the desired effect. The former is TS syntax, seemingly constructing a `NonNullable` type for you, while the latter is JS syntax. After this operation, the value of `hook` becomes `undefined | NonNullable<hook>`, and then after passing through `if`, `undefined` is filtered out.

Especially the last method seemed quite magical, originally possibly being `undefined` (I refer to it as implicit), now this "implicit `undefined`" was converted into an explicit `undefined`, which could be filtered out. Of course, this is just my description; I didn't fully understand the specific reasons, and since TS later fixed this issue, I didn't delve further into it.

### 【Resolved】distributive conditional types

The original issue can be viewed [here](https://github.com/microsoft/TypeScript/issues/42072).

This issue is actually related to the dispatcher inference mentioned above, and it was probably raised before I rewrote this part. At that time, I did not know what a [distributive conditional type](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html#distributive-conditional-types) was, and as mentioned before, the type inference of `Dispatcher` was overly complex (the front part was optimized by me later, the previous judgment was more complex). Looking at the large segments of conditional branches, I had no idea where the error originated from.

I remember using very naive methods at the time, manually breaking down the conditions and judging step by step, finally discovering the quirk. But since I didn't know the concept of Distributive conditional types and searching was quite difficult, I eventually discovered this concept and suddenly everything became clear. Below is a simple example:

```ts
type NakedExample<T> = T extends void ? "true" : "false";

type OptionalNumber = number | undefined;

// 'true' | 'false'
type Foo = NakedExample<OptionalNumber>;

// 'false'
type Bar = OptionalNumber extends void ? "true" : "false";
```

The official explanation is as follows:

> Conditional types in which the checked type is a naked type parameter are called distributive conditional types. Distributive conditional types are automatically distributed over union types during instantiation. For example, an instantiation of T extends U ? X : Y with the type argument A | B | C for T is resolved as (A extends U ? X : Y) | (B extends U ? X : Y) | (C extends U ? X : Y).

I also inquired about why it was designed this way, and [RyanCavanaugh](https://github.com/RyanCavanaugh)'s answer is as follows:

> Distributivity is usually what you want in those scenarios, so it's the default when the checked type is a naked type parameter. This was in favor of making special syntax for distributivity/non-distributivity since, with this behavior, you just generally don't have to think about it.

The general idea is that this is indeed a great design, and this design aligns with our regular thinking (which is why I didn't notice anything unusual before), and this kind of distributive generics design can be used to develop many utility types:

```ts
type Diff<T, U> = T extends U ? never : T; // Remove types from T that are assignable to U
type Filter<T, U> = T extends U ? T : never; // Remove types from T that are not assignable to U

type NonNullable<T> = Diff<T, null | undefined>; // Remove null and undefined from T

type T34 = NonNullable<string | number | undefined>; // string | number
```

Well-known types such as `Pick`, `Exclude`, etc., are all based on this design.

Having discussed so many resolved issues, there are actually many unresolved ones, some limited by my current abilities, and some by TS limitations. I would be extremely grateful if everyone could help contribute to Rematch after reading this! Let's take a look at those unresolved issues next.

### 【Unresolved】circularly reference

The original issue can be viewed [here](https://github.com/microsoft/TypeScript/issues/40279).

I previously mentioned **Circularly Reference**, mainly regarding the `Models` type. But there's another kind of circular reference [situation](https://www.typescriptlang.org/play?ts=4.4.2#code/C4TwDgpgBAogHsATgQwMbADwCUD2PhQQIQB2AJgM5RYSo6JkYVICWJA5gDRTIkgB8-KAF4oAbwBQUaVADaAawggobKIpA4AZtTzAAugC4d+BUr0SAvhImhIsTZtrAKGACpDRACjBH4SNJjuAJQiQgBuOCxk1kRg9AR0JMxQqIgQyMAQRti6hMTkVDR0DEysHNy8AvyeIcJCGFIyMHmZBfaO6C64+PycjdLVALY4RpIy44QOThQAggD8vn0TMhBTnQBCC7D9UBa1QmPLkx3OM4s746snFJu+OxYiUDWhT2BBRpUvYNZsmYiaaGg3QIh2kmjwRlsEC0UHBOEs1kSyThj1S6UyOR6NU8oOO0zOTzILAoYAyqAAFvtxBcZAB6WlQADu9EUZCZLGA5JwAFcCGQIMxENz0Gx2FAiSSyeSadIkTgADYQAB08pw7E8EtJwApSrhQQA3Pclis1s51kYcbC8LsqbjxvTYcgWIq2YyOeTxQKkMLgKKZSkcEkFcrVeq9YbxhY+nsgA):

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

As you can see, in the `effectsB` style, TS reports a circular reference error, but this is actually OK at the code level, because the writing of `effectsB` and `effectsA` is almost identical. The only difference is that `effectsA` always accesses `foo` through `dispatch`, while `effectsB` first destructures, which may cause `foo` in `effectsB` to always be `undefined`, as `foo` might be added to `dispatch` later. This problem can be avoided using the `effectsA` method. Rematch once had such an [issue](https://github.com/rematch/rematch/issues/811).

Although the above problem was resolved (the solution can be viewed in the issue link), TS still considers there to be a circular reference:

- The type of `Root['foo']` is `typeof foo`
  - To know the type of `foo`, the parameter type of `effectsB` is needed
    - The parameter type of `effectsB` is `Extract<Root>`
      - The destructured `foo` type is `Extract<Root>['foo']`, i.e., `Root['foo']`
        - Back to the first

This problem has not been resolved, and there are several similar issues, some quite strange. Interested readers can refer to some of the following comments:

- https://github.com/microsoft/TypeScript/issues/35546#issuecomment-611485331
- https://github.com/microsoft/TypeScript/issues/35546#issuecomment-645036905
- https://github.com/microsoft/TypeScript/issues/35546#issuecomment-668287404

### 【Unresolved】partial arg inference

The original issue can be viewed [here](https://github.com/microsoft/TypeScript/issues/41668).

This issue actually involves two problems, one of which is the same as mentioned in the section on the `createModel` utility function, related to [partial type parameter inference](https://github.com/microsoft/TypeScript/pull/26349). The other problem was that I thought too simply, like in the following simplified [example](https://www.typescriptlang.org/play?ts=4.4.2#code/KYDwDg9gTgLgBAE2AYwDYEMrDjAnmbAWQiVQCVgEBXZYKAZzgF44BvAWACgBIAbQGtguAFxx6MKAEsAdgHMAuqIrVaULgF8uXUJFiIUGLDnzZlNOszgAKLtzDpcqCOgSj003FwCUzAHxwANwhJBC1OJDRMbGQIaXE4ZCx0GGBiUlEAHjI4UBTpBEY04HJKcwZfKwBbCFFWOCwVOnolOHUfJn8gkLCIw2jY+MTgZNSSYoB1SRgACwgqGDNVAGUJSWQYSQHMsgrq2vrS1Wa4bLa-QODQzi4Aehu4e0dnBAfMdErGGTgAA3pgGCWEEq-2mMlk3wS7jg0gg8AARtgZAAzOiUODoRjSKiVBFqTgxOLwJEQCCWIYjIqoKwcThwA6NBi1Lh0ul3azAmYkHx-AFAkFgqyPJwuAD8bg8XlEXSuLLE-0BHNBckFDmFLxYAEYfDTZZpaa0NF4wmyhc9XlB3p9pD8eQr+XIIcgoQi4MjUS8MdDsbiuAT4nDMGSkilKZNOfNFnQVlJ1ps4tTmfSyscdSy2VUQVy5bzFQLTaLRFicXRJRduvq6ba+TM86qzZrtYm6Xrm4bjfd8y97BaPq7rb95dWleDIdaXW6GujMd6LAB3KbTHLgVBrKbGAjo6QwmDJOO+gbw9AALyDwxDYypqYayaZFbg6cVWarueVnbFXuLUFL0qb2btNdfOsXELGcoEsLU2F-FsDU4NouCAA):

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

In `foo` from the example above, I originally thought using the default parameter `payload = 1` in `setSomething` could infer `payload` as `number`. I later realized that since we have type constraints on `reducers` in `createModel`, whether using default parameter assignment or explicitly declaring the parameter type, we need to ensure that this type is compatible with the constrained type. But default assignment does not change the inferred type.

Actually, in Rematch's code, the type of `Reducer` is as follows:

```ts
export declare type Reducer<TState = any> = (
  state: TState,
  payload?: any
) => TState;
```

As mentioned earlier, since partial type inference is not possible (i.e., `TState` uses the user-defined generic, while `payload` is inferred), we defined `payload` as an optional `any` type. This way, since any type is compatible with `any`, users can narrow down its type when actually defining `payload`.

> The optional and mandatory effects of `payload` above are the same, because `undefined | any` equals `any`.

### 【Unresolved】Type Design for Reducer and Effect with the Same Name

As mentioned earlier, in Rematch, `reducer` can be named the same as `effect`. Moreover, when called, the `reducer` is executed first, followed by the `effect`. I'm not sure why it was designed this way initially, and this behavior is hidden from users.

In addition, this posed a significant challenge to type design, or even it was impossible to achieve. Below is a code snippet from the effect middleware:

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

The above code first determines whether `action.type` exists in `effects`. If so, it calls the `reducer` first (`next(action)` represents calling the next middleware, with the reducer being executed last).

So, how should we consider this at the type level? Use a union to suggest two types? Or consider providing only the reducer type? I don't think either solution is quite reasonable, although [Sergio](https://github.com/semoal) is currently using solution 2, which was mentioned earlier when discussing RematchDispatch. Let's [review it here](https://www.typescriptlang.org/play?#code/JYWwDg9gTgLgBAbzgWQKZQOaoKIA8DGANgK4DOwAbqnAL5wBmUEIcA5DAJ5ioC09qpGKwBQwztzgAlVABNi+dAEF8MYBAB2pOAF5EwuAbjB1+KKhCp1MAFxwAFHbABDDoQhOZt9cRAAjdACUOgB8cBQQwDJBAGR6hvFGpNj0-Cq29E6EpKgA3PqGNHnxMqim5pY29o4ubh62glDGGEHaoeGRMXEJBsBJKaWVGVm5+QaFo0YmZhZWAMIQYBy2Ds6u7p5w3n7ocAA+cA1NLW0RUXCxCBPxvcmpg5nZRQV542Jc1LcDyqoaWrqX1ym5Ssy2qazqmx8-igewOMEa6maIUh2ygnQB3USnzScHhxBG8XGxVK0wqoNWtQ2W2hxxRNPOXW6N36OLxBOeE2MZRmMHmi3JNXWXihgWR1NFFyuhmZd1sbKeYxeeTeEm+ak0ACFUPRoNRdNI5AooGrfgzsTATZo8uJqJbSIp6DAdro0JgcAQSOQqAAeA3yJQqdWkAA0cHNduCypKRCcZjg+F+8Ccgd+Wp1ZlsdrTury0cIseoCc0SZTmgdTqgmdL9sd6GVwgA9A24AAVAAWvTgqFwYDMpHIGkSmwg8HwmXzvkIqAAdI3mwYW+82OpUFQoKw4G2nFp1BB4+ODsAMOonDBiH3p3YAEwAZgALABOALCZM-TXa3XTrmkqx2ACMATKk29i9gs6CcEE37ApUYKUsKqK0uKML7O0MgvtW5boF+QI8v+z5znAiiYD4FRwBA9C4kurBIRuna7km-ZHiek7UDAe7OFATgWBWZEUTay6rugrCXred4AKzPq+QaYVA07Rj+MB4UBzZERgJFWLxlESKwhyIrRO4jnA27kMeTgsbi7GxlxqA8eRWnUNRIrriJ94Sehb41hW2HchUfIcHYrB-qw+FAA):

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

Due to the setting of `isEffect: boolean`, the same name reducer and effect lead to the appearance of the `never` type, making the function uncallable. So we switched to the [`MergeExclusive`](https://github.com/sindresorhus/type-fest/blob/main/source/merge-exclusive.d.ts) utility type for this task, which is relatively easy to understand. For example, for two types `A` and `B`, the end result is using union to replace intersection, but before the union, it does two things to `A` and `B`:

- For `A`, set all attributes different from `B` to `?: never` (since optional is equivalent to `undefined`, here it's actually `undefined | never`, i.e., `undefined`)
- Then intersect with `B`

Do the same for `B` and then unite them.

Returning to the front, what are the advantages and disadvantages of this method? The advantage is that when reducer and effect have the same name but completely different `payloads`, the use of a union type will cause `payload` to be `never`, thus uncallable, as seen in `actionsAfter.decrement(1)` above. Why is this an advantage? Because it perfectly matches Rematch's design, as the action processed first by the reducer will continue to be passed to the effect. If the `payload` types of the two are completely different, it obviously could lead to errors.

What about the disadvantages? The disadvantage is also due to the union type. Look at `actionsAfter.increment(1)` above. The `payload` type of the reducer is a subset of the effect's `payload` type, which is expected. For example, it will prompt `payload: number`, allowing the code to execute normally. But if reversed, TS can still ensure the successful execution of the code. However, since the reducer is executed first, the intention is to prompt the reducer type, but here it prompts the effect type, as seen in `actionsAfter.incrementCopy('1')` above.

Honestly, reducers and effects with the same name are indeed strange, and we should avoid this situation.

### 【Unresolved】`this` types

The original issue can be viewed [here](https://github.com/rematch/rematch/issues/870).

In the [third article](https://juejin.cn/post/6992935103286640671#heading-5) of this column, when discussing Rematch's core plugins, I mentioned that the context `this` in the effect function is bound to `dispatch[modelName]`. This approach conveniently allows the use of `this` in the effect to dispatch all actions of the current model. However, this also poses challenges to type compatibility at the TypeScript (TS) level. The current definition of the effect type is as follows:

```ts
export type ModelEffect<TModels extends Models<TModels>> = (
  this: ModelEffectThisTyped,
  payload: Action["payload"],
  rootState: RematchRootState<TModels>,
  meta: Action["meta"]
) => any;
```

If it were changed to:

```ts
export type ModelEffect<TModel, TModels extends Models<TModels>> = (
  this: ModelDispatcher<TModel>,
  payload: Action["payload"],
  rootState: RematchRootState<TModels>,
  meta: Action["meta"]
) => any;
```

In that case, almost all types would need to add the `TModel` generic parameter, and it would also cause `Model` to become like this:

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

This leads to `Model` also circularly referencing itself, similar to the **Circularly Reference** section mentioned earlier with `Models`. And as I mentioned, perhaps we can use `this` to represent the type itself in the type definition. If feasible, I think this approach can also be applied to `Model`. Currently, I do not have much time, but I will continue to pay attention to these two issues.

### 【Unresolved】select plugin types

The last unresolved issue is how to perfect the type definition for the select plugin. The code written by the original author of this plugin is quite complex, even more so than the [reselect](https://github.com/reduxjs/reselect) source code, which I find simpler. I cannot fully comprehend it, hence the type definition of this plugin is only slightly improved, and many parts are not thoroughly worked out. If anyone is interested, it would be great to delve into it and possibly fix it.

## Summary

In fact, at the beginning of the reconstruction of the Rematch type system, my skill level was quite limited, so you will see that many of my designs were coincidental or accidental discoveries, just finding that they worked, which felt quite miraculous. But through writing this article, I traced back many of these "strange" designs and discovered more interesting aspects of TS. I hope everyone learns not only what things are but also understands why they are the way they are, and keeps the passion for learning!

This concludes this column. I hope that by reading all the articles, everyone can gain a deeper understanding of Rematch, thereby efficiently developing and mastering state management.
