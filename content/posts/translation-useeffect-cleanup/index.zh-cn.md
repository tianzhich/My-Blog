---
title: "【译】useEffect 的清除函数以及它的两种调用条件"
date: 2023-08-21T14:21:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["React", "react-training", "翻译"]
categories: ["编程"]
series: ["translations", "react-training-blog"]
featuredImage: "https://reacttraining.com/images/blog/useEffect-cleanup.jpg"
---

本文翻译自：[https://reacttraining.com/blog/useEffect-cleanup](https://reacttraining.com/blog/useEffect-cleanup)

<!--more-->

---

React 的 `effect` 函数中会返回一个清除函数。在组件卸载时这个函数会被执行，这一点你可能已经了解了。

在我的讲习班上，我问开发者们这个函数什么时候会被调用，我通常会得到上面这一个答案。但是在我过去几年教过的 100 多个班上，我只遇到过一个人回答了完整答案（向他致敬）。


```ts
useEffect(() => {
  getUser(userId).then((user) => {
    setUser(user)
  })

  // 清除函数: 组件卸载时被调用
  return () => {}
}, [userId])
```

你可能在略读这篇文章，想直接知道它的第二种调用条件：

当 `effect` 的依赖项发生变化时，需要重新执行时，清除函数也会被调用。但是在下一个 effect 执行前，前一个 effect 的清除函数会先被调用。也许你需要读完整篇文章才能真正理解。。。

## 为什么需要清除函数

为了更好理解这两种调用条件，我们首先需要解释*为什么*你需要清除函数。

不幸的是，上面的代码中最尽人皆知的原因也是最容易误导人的，因此我们从这段代码开始。人们认为需要清除函数的理由是，如果不这样做，我们可能会“在一个已卸载的组件上进行 setState”。

在我们的[另一篇文章](https://reacttraining.com/blog/setting-state-on-unmounted-component)中，已经深入解释过你不用去关注这个特定问题，但这个问题是一个好的切入点，我们需要修复这个问题，稍后会说明其他原因。

当组件卸载时，阻止 setState 的示例代码如下：

```ts
useEffect(() => {
  // 1. 组件渲染后，useEffect 函数被执行
  // 此时可以确保组件已渲染，因此我们设置这个标识
  let isMounted = true
  getUser(userId).then((user) => {
    if (mounted) {
      setUser(user)
    }
  })

  // 2. 我们执行了副作用（getUser），现在我们需要
  // “清除”由于这个副作用而引发的潜在问题。
  return () => {
    isMounted = false
  }
}, [userId])
```

目前为止的重点是，我们返回了这个清除函数，但没有调用它，且上面的 promise 仍然处于 pending 状态。

现在，竟态条件（race condition）出现了。如果组件在 promise 解析之前就被卸载呢？（比如我们可能会跳转到其他页面），或者是 promise 先解析呢？

如果组件先卸载，上面这段代码会阻止我们在一个被卸载的组件上执行 setState -- 如果这正是你想要的话。[我说过其实这无关紧要](https://reacttraining.com/blog/setting-state-on-unmounted-component)。

确切地说，我确实需要上面这个清除函数的解决方案，但只是因为它修复了另外一个问题，由另一种竟态条件导致的问题。

## 竟态条件

让我们回到最初有 `isMounted` 的那段代码：

```ts
function UserProfile({ userId }) {
  const [user, setUser] = useState(null)

  useEffect(() => {
    getUser(userId).then((user) => {
      setUser(user)
    })
    return () => {}
  }, [userId])

  return <div>...</div>
}
```

在 `UserProfile` 组件中，让我们假设可以点击这个 User 的朋友链接。当我们在浏览 `users/1` 时，我们可以快速点击跳转到 `users/2`，然后 `users/3`、`users/4`，最后 `users/5`。所有这些点击都会导致组件以一个新的 `userId` 属性重新渲染。

我们快速点击，因为 `users/5` 最后点击，他应该是最后看到的一个，但结果也许是这样的：

每次我们点击时，组件会重新渲染，effect 会接收一个新的 `userId`，重新执行。当出现竟态条件时，网络返回的结果可能和请求的顺序不一致。最后看到的是网络最后返回的。我们想要看到 `users/5` 但可能 `users/4` 的请求最慢，最后被解析，你最后也会看到错误的 User 4。

当“切换” effects 时，清除函数也会被调用。换句话说，就是在依赖项变化，重新执行 `useEffect()` 函数前，React 会先执行上一个 effect 的清除函数。

从概念上讲，这就是 React 调用函数式组件的情况，让我们假设正在执行的 effect 为“current effect”：

```ts
UserProfile() // useEffect runs based on user 1: this is the current effect
```

假设接下来组件被重新渲染（`userId` 外的变化导致）。React 在再次用函数式组件，”current effect“ 仍属于第一次渲染：

```ts
UserProfile() // <-- current effect
UserProfile() // re-render
UserProfile() // re-render
```

你可以看到，有“recent render”，但是“current effect”来自于上一次渲染。

然后 `userId` 更新，组件再次重新渲染。这次重新渲染需要重新执行 `useEffect()`。但在此之前，我们需要先“清除”旧的“current effect”。

```ts
UserProfile() // 1. cleanup this effect first
UserProfile() // re-render
UserProfile() // re-render
UserProfile() // 2. run this effect after the previous cleanup runs
```

如果我们像下面一样清除，就可以修复竟态：

```ts
useEffect(() => {
  let isCurrent = true
  getUser(userId).then((user) => {
    if (isCurrent) {
      setUser(user)
    }
  })
  return () => {
    isCurrent = false
  }
}, [userId])
```

看吧，和上面提到的为了避免在已卸载组件上 setState 的解决方案完全一致，除了变量命名更准确，毕竟我们现在知道清除函数并非只在组件卸载时被调用。

使用了这样的清除函数，每次 `userId` 更新时，我们都会先清除**上一个** effect，这避免了当 promise 解析时，仍然执行 setState。然后执行下一个 `userId` 关联的 effect，同时将它变成“current effect”。

当点击很快时，我们更新“current effect”，之前的 promise 仍然处于 pending 态，最终的结果是我们通过阻止在之前的 effect 里执行 setState，来避免竟态。我们只想当 `users/5` 解析时看到 User 5，至于之前的无论是否解析，也不会因为调用了 setState 而导致最终结果出错。

有趣的时，这种方案也避免了在一个卸载的组件上执行 setState -- [尽管这不是我们所关注的](https://reacttraining.com/blog/setting-state-on-unmounted-component)，但也很有趣。

在某种程度上，你可以将这两种条件抽象成一条规则：

当 effect 不再相关时，执行清除函数。当组件卸载，或者是我们需要抛弃一个旧的 effect 来创建一个新的时，都意味着不再相关。不管你怎么思考，只要能够理解，我都 OK。

祝编码愉快。
