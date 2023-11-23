---
title: "Rematch Code Deep Dive: Extra Edition - Dive into Rematch Dispatcher type"
date: 2023-11-23T16:33:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["rematch", "redux", "source code", "TypeScript"]
categories: ["Programming"]
series: ["rematch-code"]
series_weight: 8
featuredImage: "featured-image.webp"
---

This is Rematch Code Deep Dive Series, an extra edition, discussing the type implementation approach for Rematch Dispatcher.

<!--more-->

Recently, I fixed a type related [bug](https://github.com/rematch/rematch/issues/936) in Rematch. Through this bug, I discovered several type bugs still present in Rematch Dispatcher, and there was also redundancy in code organization. Taking this opportunity, I [refactored](https://github.com/rematch/rematch/pull/937) this part and found some interesting aspects.

## **Issue Reproduction**

Below is a simple reproduction example, which you can also click to view in the [Playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAsiBKEAmBXAxhATlAvFAFAM7ACGwEAXFAHYoC2ARlgDRRgkgA2A9iUlSWogAlLgB8UAG7cAlkgDcAWABQKiAA8w3TMCihIUAKLrgmEmmAARGYXbA0ACywAxTNzqJUGTAB4AKp7oWBI4KlBQAchB2Brk1EiEBMRklDT0TJisAHQ5JJgA5oRUMtQAZlgRiMSiOBLScmFQAPyVEMRQsRDxiQDaALqNLYHe1rZkjliNVJHEPQCMfR0mXQlQKPEQpSXIg1DDWKN2E74zwD0ADH1iU3tRIzZHTidVZ5esp-NXN9QQkpOqyg0Wh0enA0H2mEO4ye-gAChweHxcFJZEh3jAIKRkfUkCEoD1Gn54VxeGjCRjSCpFp1uvicawcQNlOEWvh8DU6qjhDcen4KSRqctaT1Gbt1khNtskEs4qsiQjSbs2ewSXwmtNiYikByUXJucyoFRlQq+BqTdrxLrtTzxZKfmi1hstvbBbLevLVQ6+ZiBUr8CqteqIprSaw6D6g97SDqcfrwlRbc7kDKVokoyQ-QHSWbPWGI9N+TGuTdjZ6c1q86QCz6i3qVCp9NA4BCoRYnsjjKZzFYHtCXG4PHcsD5m0PMGJ5FAAPRTgi1pD15SNqAAQSEndTADVUcjBCAU7Scc0oHNDVBzpOZ+eoAAfE8NsGr9dChIAOV+FTwe4Pqx+f2wLSnlQF7TrO5y3veALLmuIArpgZggBu3TbnIcEIciPR7q6qZ0qiiyAWeIFXnMD4GDBaEcEhb4fpgFH7ngmFCNhwp-lg+EnoRl5gYuKhAA).

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

In `ExtractDispatcherFromReducer`, I intended to use `TRest[0]` to get the payload parameter, and `TRest[1]` to get the meta parameter. And I thought when no payload or meta parameter was defined, the corresponding generic parameter in the selected `ReducerDispatcher` would be set to the default value `void`. However, the reality was different from what I expected; the final parameter turned out to be `undefined`. And because `[undefined] extends [void] === true`, the effect was the same as `[void] extends [void] === true`, so this bug remained undiscovered.

However, once the payload type is `any`, this bug could no longer hide. Since `[any] extends [void] === true`, it would ultimately enter the `() => void` branch, where the payload parameter deduction would be empty, contradicting the definition.

>  `any extends void === true | false`.

So, how to solve this problem? We just need to find a type `T` such that `[any] extends [T] === false`. Yes, we can use `never` to replace `void`.

> `any extends never === true | false`。

That's where this particular bug is solved. But two problems remain. Firstly, as mentioned earlier, when the meta parameter is not defined, `TRest[1]` taken and passed into `ReducerDispatcher` is always `undefined`, not triggering the default assignment of `never`, thus always deducing the parameter to include an optional meta. We can optimize `ReducerDispatcher` as follows:

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

Due to the addition of a new branch judgment, `TRest[1]` is no longer passed into `ReducerDispatcher`, allowing `TMeta` to be defaulted to `never`.

The second problem is that we cannot distinguish between a user-defined type of `undefined` and optional parameters. Once a user defines the payload or meta type as `undefined | T` (although doing so makes no sense, as the optional symbol `?` could be used directly), the above type gymnastics will fail, appearing as optional parameters even if the user defined `undefined | T` type.

Initially, in the design of types, regardless of whether the user defined optional parameters or explicitly `undefined` type, the Rematch Dispatcher deduced all as optional parameters after inference. This approach did not lead to runtime errors, and no users have reported related issues. However, this approach is not consistent with the behavior of TS. Out of a desire for continuous improvement, I plan to distinguish between these two this time.

## **Distinguishing Between `Undefined` Type and Optional Parameters**

Once we can distinguish between a user-defined type of `undefined` and the form of optional parameters through gymnastics, we can achieve perfect deduction. The challenge here lies in how to retain the optional nature of parameters while extracting them. Once we use indexing to access specific parameters, their optional nature disappears:

```ts
type MyReducer = (state: number, payload?: number) => void;

type ParamPayload = Parameters<MyReducer>[1]; // number | undefined
```

### **Preserving the Array Structure of Optional Parameters**

This gave me an idea: if we want to retain the optional nature of parameters, we cannot extract them individually; instead, we must retain the outer array structure:

```ts
type Params = Parameters<MyReducer>; // [state: number, payload?: number | undefined]
```

Having retained the optional nature of parameters, we then proceed to judge and extract.

### **Judging and Extracting Optional Parameters**

The following two rules can help us sort out our thoughts:

1. Arrays with more parameters cannot be assigned to those with fewer, e.g., the result of `[payload: unknown, meta?: unknown] extends [payload: unknown] ? 1 : 0` will be 0.
2. Arrays with mandatory parameters can be assigned to those with optional parameters, e.g., the result of `[payload: unknown] extends [payload?: unknown] ? 1 : 0` is 1, but the reverse is not true.

If using `Parameters` or `infer` to obtain parameter types, the second point above is valid, but if written directly in array form, it is not, as seen in the example below:

```ts
type Func = (p?: unknown) => void;
type ParameterType = Parameters<Func>;

type case1 = [p?: unknown] extends [payload: unknown] ? 1 : 0; // 1
type case2 = ParameterType extends [payload: unknown] ? 1 : 0; // 0
```

I'm not sure if the above is a bug, and I haven't found an answer yet. However, since the parameter arrays in Rematch all use `infer` for extraction, there is no impact for now.

Using the above logic, we can first judge the situation with fewer parameters (first point), and then continue to deduce the situation with mandatory parameters (second point). Let's look at the conditional judgments for reducer and effect in Rematch, respectively.

### **Extracting Parameters from `Reducer`**

Since we do not need the first parameter `state` of the reducer, we first ignore it, then pass all remaining parameters into our newly added type `ExtractParametersFromReducer` for processing. As a result, the previously mentioned `ExtractDispatcherFromReducer` type can also be simplified, as we pass the parameters to the specialized `ExtractParametersFromReducer` for processing, making it very concise.

Additionally, the `RematchDispatcher` type in Rematch has also been greatly simplified, as in the previous logic, both `RematchDispatcher` and `ExtractDispatcherFromReducer` judged parameters, leading to redundancy in the code. Now, the input and output of the `ExtractParametersFromReducer` type are both parameter arrays; no preprocessing is required for the input, and the output can be used directly. For details, you can click this [PR](https://github.com/rematch/rematch/pull/960) to view.

Below is the related code:

```ts
// Extract remaining parameters other than `state`
export type ExtractRematchDispatcherFromReducer<TState, TReducer> =
  TReducer extends (state: TState, ...args: infer TRest) => TState | void
    ? RematchDispatcher<false, ExtractParametersFromReducer<TRest>>
    : never;

// Judging and handling optional parameters
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

Careful readers can see that in `ExtractParametersFromReducer`, in addition to `infer TPayload`, I also have `infer TPayloadMayUndefined`. This is because when a user defines a parameter as `p: number | undefined`, the `TPayload` type only has `number`, and `undefined` is ignored. Therefore, here we use `TPayloadMayUndefined` to correctly deduce this situation. (The same applies to `TMetaMayUndefined` below.)

### **Extracting Parameters from `Effect`**

In Rematch's `effects`, the second parameter `rootState` is the one we need to ignore. Since it's not convenient to directly remove the second parameter type, we pass all the parameters to the corresponding `ExtractParametersFromEffect` for processing. The code is as follows:

```ts
// Due to the inconvenience of directly removing the second parameter, all parameters are extracted and handed over to ExtractParametersFromEffect for processing.
export type ExtractRematchDispatcherFromEffect<
  TEffect extends ModelEffect<TModels>,
  TModels extends Models<TModels>
> = TEffect extends (...args: infer TRest) => infer TReturn
  ? RematchDispatcher<true, ExtractParametersFromEffect<TRest>, TReturn>
  : never;

// Judging and Handling Optional Parameters
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

The situation with `effect` is roughly similar to `reducer`. In cases containing only `payload`, we define the second parameter `s` as optional in the condition, so it can hit this branch whether it's defined or not. Then, for situations with three or more parameters, we first judge for mandatory and finally for optional, consistent with `reducer`.

> When `effect` was initially designed, considering that `payload` is a frequently used parameter, it was placed first, with `rootState` in the second position. Subsequently added `meta` was naturally put in the third position, and `meta` is less frequently used. For `reducer`, the current `model`'s `state` generally has a higher frequency of access, so it's placed first, while `payload` and `meta` are in the second and third positions, respectively.

## Conclusion

When dealing with such gymnastics problems, it's important to first think about whether there are constraints that can gradually narrow down the possible range. For example, by the rule that "arrays with more parameters cannot be assigned to those with fewer," we can use the constraint of "arrays with fewer parameters" to first handle cases with fewer parameters, and then gradually deal with cases with more parameters. Within each case, we continue to use the rule that "arrays with optional parameters cannot be assigned to those with mandatory parameters," using the constraint of "arrays with mandatory parameters" to first handle mandatory parameter situations.

Also, for multiple parameters, since [TS requires optional parameters to be placed after mandatory ones](https://stackoverflow.com/q/46958782/14251417), we prioritize handling the situation where the optional parameters are at the end, and finally deal with the situation where all parameters are optional.

Once you find such patterns, you are a TypeScript magician🏆!
