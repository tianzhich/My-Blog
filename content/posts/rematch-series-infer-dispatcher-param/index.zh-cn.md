---
title: "【Rematch 源码系列】番外一、详解 Rematch Dispatcher 类型体操"
date: 2021-09-08T12:12:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["rematch", "redux", "源代码", "TypeScript"]
categories: ["编程"]
series: ["rematch-code"]
series_weight: 8
featuredImage: "featured-image.webp"
---

Rematch 源码解读系列，番外的第 1️⃣ 篇，和大家聊下 Rematch Dispatcher 的 TypeScript 类型实现思路。

<!--more-->

前段时间修复了 Rematch 的[一个类型 bug](https://github.com/rematch/rematch/issues/936)。通过这个 bug，又发现了 Rematch Dispatcher 尚存的几个类型 bug，且代码组织上也有冗余。借这个机会，我[重构](https://github.com/rematch/rematch/pull/937)了这部分，并且发现了一些有意思的部分。

## 问题复现

下面是一个简单的复现例子，你也可以点击 [Playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAsiBKEAmBXAxhATlAvFAFAM7ACGwEAXFAHYoC2ARlgDRRgkgA2A9iUlSWogAlLgB8UAG7cAlkgDcAWABQKiAA8w3TMCihIUAKLrgmEmmAARGYXbA0ACywAxTNzqJUGTAB4AKp7oWBI4KlBQAchB2Brk1EiEBMRklDT0TJisAHQ5JJgA5oRUMtQAZlgRiMSiOBLScmFQAPyVEMRQsRDxiQDaALqNLYHe1rZkjliNVJHEPQCMfR0mXQlQKPEQpSXIg1DDWKN2E74zwD0ADH1iU3tRIzZHTidVZ5esp-NXN9QQkpOqyg0Wh0enA0H2mEO4ye-gAChweHxcFJZEh3jAIKRkfUkCEoD1Gn54VxeGjCRjSCpFp1uvicawcQNlOEWvh8DU6qjhDcen4KSRqctaT1Gbt1khNtskEs4qsiQjSbs2ewSXwmtNiYikByUXJucyoFRlQq+BqTdrxLrtTzxZKfmi1hstvbBbLevLVQ6+ZiBUr8CqteqIprSaw6D6g97SDqcfrwlRbc7kDKVokoyQ-QHSWbPWGI9N+TGuTdjZ6c1q86QCz6i3qVCp9NA4BCoRYnsjjKZzFYHtCXG4PHcsD5m0PMGJ5FAAPRTgi1pD15SNqAAQSEndTADVUcjBCAU7Scc0oHNDVBzpOZ+eoAAfE8NsGr9dChIAOV+FTwe4Pqx+f2wLSnlQF7TrO5y3veALLmuIArpgZggBu3TbnIcEIciPR7q6qZ0qiiyAWeIFXnMD4GDBaEcEhb4fpgFH7ngmFCNhwp-lg+EnoRl5gYuKhAA)。

```ts
type MyReducer = (state: number, payload: any) => void;

export type ExtractDispatcherFromReducer<TReducer> = TReducer extends (
  state: number,
  ...args: infer TRest
) => void
  ? TRest extends []
    ? ReducerDispatcher
    : ReducerDispatcher<TRest[0], TRest[1]>
  : never;

export type ReducerDispatcher<TPayload = void, TMeta = void> = [
  TPayload,
  TMeta
] extends [void, void]
  ? () => void
  : [TMeta] extends [void]
  ? undefined extends TPayload
    ? (payload?: TPayload) => void
    : (payload: TPayload) => void
  : [undefined, undefined] extends [TPayload, TMeta]
  ? (payload?: TPayload, meta?: TMeta) => void
  : undefined extends TMeta
  ? (payload: TPayload, meta?: TMeta) => void
  : (payload: TPayload, meta: TMeta) => void;

type MyReducerDispacther = ExtractDispatcherFromReducer<MyReducer>; // () => void
```

在 `ExtractDispatcherFromReducer` 里，我是打算使用 `TRest[0]` 获取到 `payload` 参数，`TRest[1]` 获取到 `meta` 参数。且以为**当没有定义 `payload` 或者 `meta` 参数时，选中的 `ReducerDispatcher` 相应泛型参数会被设置为默认值 `void`**。然而事实与我想的不一样，最终传入的参数其实会是 `undefined`。且因为 `[undefined] extends [void] === true`，效果上来看和 `[void] extends [void] === true` 并无不同，所以这个 bug 也一直没被发现。

不过，一旦 `payload` 类型为 `any`，这个 bug 便无处藏身了，由于 `[any] extends [void] === true`，最终进入到 `() => void` 这个分支，此时 `payload` 参数推导为空，和定义不符。

> 注意：`any extends void === true | false`。

那这个问题如何解决？其实只需找到一个类型 `T` 使得 `[any] extends [T] === false`。没错，我们可以使用 `never` 来替换 `void`。

> 注意：`any extends never === true | false`。

到这里为止，上面这个特定的 bug 就解决了。但还遗留两个问题，首先是前面提到的，当没有定义 `meta` 参数时，`TRest[1]` 取到并传入 `ReducerDispatcher` 的永远是 `undefined`，并不会触发默认赋值 `never`，这样一来推导的参数始终包含可选的 `meta`。我们可以对 `ReducerDispatcher` 优化如下：

```ts
export type ExtractDispatcherFromReducer<TReducer> = TReducer extends (
  state: number,
  ...args: infer TRest
) => void
  ? TRest extends []
    ? ReducerDispatcher
    : TRest[1] extends undefined
    ? ReducerDispatcher<TRest[0]>
    : ReducerDispatcher<TRest[0], TRest[1]>
  : never;
```

由于新增了一个分支判断，此时 `TRest[1]` 不传入 `ReducerDispatcher`，`TMeta` 可以被默认赋值为 `never`。

第二个问题则是我们无法区分用户自定义的 `undefined` 类型，一旦用户定义 `payload` 或 `meta` 的类型为 `undefined | T`（虽然这样做毫无意义，因为可直接使用可选符号 `?`），那么上面的类型体操也将失效，表现为即使用户定义了 `undefined | T` 类型，推导出来的参数会变成可选参数。

最开始的类型设计中，为了简化推导，不管用户定义的是可选参数还是明确的 `undefined` 类型，Rematch Dispatcher 推导后的均为可选参数。这样做并不会导致 runtime error，且也没有用户反馈过相关问题。不过这样做和 TS 的行为并不一致，出于精益求精的考量，我打算这次想办法对这俩进行区分。

## 区分 `undefined` 类型和可选参数

一旦我们可以用体操将用户自定义的 `undefined` 类型和可选参数形式区分开来，那么我们便可以做到完美的推导。这里的难点在于如何在提取参数的同时保留参数的可选特性。一旦我们使用索引来访问具体参数，它的可选特性便会消失：

```ts
type MyReducer = (state: number, payload?: number) => void;

type ParamPayload = Parameters<MyReducer>[1]; // number | undefined
```

### 保留可选参数的数组结构

这也给了我启示，如果想要保留参数的可选性质，则不能单独进行提取，而是要保留外面的数组结构：

```ts
type Params = Parameters<MyReducer>; // [state: number, payload?: number | undefined]
```

保留了参数的可选性质，接下来我们要进行判断和提取。

### 判断和提取可选参数

下面 2 条规则可以帮助我们梳理思路：

1. 参数多的数组无法赋值给参数少的，例如 `[payload: unknown, meta?: unknown] extends [payload: unknown] ? 1 : 0` 的结果将会是 `0`。
2. 含有必选参数的数组可以赋值给含有可选参数的数组，例如 `[payload: unknown] extends [payload?: unknown] ? 1 : 0` 的结果是 `1`，但反之不成立。

**注意：如果使用 `Parameters` 或 `infer` 来获取参数类型，上面第 2 点是成立的，但如果直接写成数组形式，则不成立，见下方例子**

```ts
type Func = (p?: unknown) => void;
type ParameterType = Parameters<Func>;

type case1 = [p?: unknown] extends [payload: unknown] ? 1 : 0; // 1
type case2 = ParameterType extends [payload: unknown] ? 1 : 0; // 0
```

不知道上面是否是一个 BUG，我也还没找到答案。不过 Rematch 里参数数组均使用 `infer` 提取，因此暂无影响。

使用上面的思路，我们可以先通过条件类型先判断参数少的情况（第一点），进而继续判断出必选参数的情况（第二点）。下面分别看看 Rematch 中针对 `reducer` 和 `effect` 的条件判断。

### 从 `reducer` 中提取参数

由于我们并不需要 `reducer` 的第一个参数 `state`，因此先将它忽略，然后把剩余参数全部传入我们新增的一个类型 `ExtractParametersFromReducer` 中处理即可。这样一来前面提到的 `ExtractDispatcherFromReducer` 类型也可以得到简化，因为我们把参数交给专门的 `ExtractParametersFromReducer` 处理，它将会变得很简洁。

此外，Rematch 中的 `RematchDispatcher` 类型也得到了极大简化，因为在之前的逻辑中，`RematchDispatcher` 和 `ExtractDispatcherFromReducer` 都对参数进行了判断，代码是冗余的，而如今类型 `ExtractParametersFromReducer` 的输入输出均为参数数组，输入无需提前处理，输出亦可以直接使用。详情可以点击[这一条 PR](https://github.com/rematch/rematch/pull/960) 查看。

下面是相关代码：

```ts
// 提取除了 `state` 外的剩余参数
export type ExtractRematchDispatcherFromReducer<TState, TReducer> =
  TReducer extends (state: TState, ...args: infer TRest) => TState | void
    ? RematchDispatcher<false, ExtractParametersFromReducer<TRest>>
    : never;

// 判断和处理可选参数
type ExtractParametersFromReducer<P extends unknown[]> = P extends []
  ? []
  : P extends [p?: infer TPayload]
  ? P extends [infer TPayloadMayUndefined]
    ? [p: TPayloadMayUndefined]
    : [p?: TPayload]
  : P extends [p?: infer TPayload, m?: infer TMeta, ...args: unknown[]]
  ? P extends [
      infer TPayloadMayUndefined,
      infer TMetaMayUndefined,
      ...unknown[]
    ]
    ? [p: TPayloadMayUndefined, m: TMetaMayUndefined]
    : P extends [infer TPayloadMayUndefined, unknown?, ...unknown[]]
    ? [p: TPayloadMayUndefined, m?: TMeta]
    : [p?: TPayload, m?: TMeta]
  : [];
```

细心的同学可以看到我在 `ExtractParametersFromReducer` 中除了 `infer TPayload` 还有一个 `infer TPayloadMayUndefined`，这是因为如果用户定义的参数为 `p: number | undefined` 时，`TPayload` 类型只有 `number` 而 `undefined` 被忽略掉了。因此这里我们使用 `TPayloadMayUndefined` 来正确推导这种情况。（下面的 `TMetaMayUndefined` 也是一样）

### 从 `effect` 中提取参数

Rematch 的 `effects` 中我们需要忽略的是第二个参数 `rootState`。由于不太方便直接去掉第二个参数类型，因此我们把所有参数全部交给对应的 `ExtractParametersFromEffect` 处理。代码如下：

```ts
// 由于不方便直接移除第二个参数，因此提取全部参数，交给 `ExtractParametersFromEffect` 处理
export type ExtractRematchDispatcherFromEffect<
  TEffect extends ModelEffect<TModels>,
  TModels extends Models<TModels>
> = TEffect extends (...args: infer TRest) => infer TReturn
  ? RematchDispatcher<true, ExtractParametersFromEffect<TRest>, TReturn>
  : never;

// 判断和处理可选参数
type ExtractParametersFromEffect<P extends unknown[]> = P extends []
  ? []
  : P extends [p?: infer TPayload, s?: unknown]
  ? P extends [infer TPayloadMayUndefined, ...unknown[]]
    ? [p: TPayloadMayUndefined]
    : [p?: TPayload]
  : P extends [
      p?: infer TPayload,
      s?: unknown,
      m?: infer TMeta,
      ...args: unknown[]
    ]
  ? P extends [
      infer TPayloadMayUndefined,
      unknown,
      infer TMetaMayUndefined,
      ...unknown[]
    ]
    ? [p: TPayloadMayUndefined, m: TMetaMayUndefined]
    : P extends [infer TPayloadMayUndefined, unknown?, unknown?, ...unknown[]]
    ? [p: TPayloadMayUndefined, m?: TMeta]
    : [p?: TPayload, m?: TMeta]
  : [];
```

`effect` 的情况和 `reducer` 大致相同，在只含 `payload` 的情况下，第二个参数 `s` 我们在条件中定义为可选即可，这样不管它是否被定义均能命中该分支。然后是三个参数及以上的情况，先判断必选，最后判断可选，和 `reducer` 一致。

> `effect` 在最初设计时，考虑到 `payload` 是个需要频繁使用的参数，因此把他放到了第一位，而 `rootState` 放到第二位，后面增加的 `meta` 则自然放入了第三位，且 `meta` 使用频率不高。对于 `reducer`，当前 `model` 的 `state` 一般而言访问频率更高，因此将它放到第一位，而 `payload` 和 `meta` 则分列二三位。

## 总结

在处理这种体操问题时，重要的是先想到是否有约束可以逐渐缩小可能的范围。比如通过「参数多的数组无法赋值给参数少的」这条规则，我们便可以使用「参数少的数组」这条约束来先处理参数少的情况，再逐步处理参数多的情况。而在各自内部，我们继续通过「含有可选参数的数组无法赋值给含有必选参数的」这条规则，使用「必选参数的数组」这条约束来先处理必选参数的情况。

同时，对于多个参数，由于 [TS 中可选参数只能位于必选参数之后](https://stackoverflow.com/q/46958782/14251417)，我们优先处理末尾参数可选的情况，最后再处理所有参数都可选的情况。

只要找到了这样的规律，你就是这类体操的冠军 🏆！
