---
id: guides-ready-state
title: Ready State
layout: docs
category: Guides
permalink: docs/guides-ready-state.html
next: guides-mutations
---

当Relay获取数据的时候，了解具体正在发生的事情会对我们有用。例如，我们想记录获取数据使用了多少时间，或者是记录错误信息到服务器。这些事件在大多数的Relay API上通过　｀onReadyStateChange｀回调提供。

## `onReadyStateChange`

当Relay获取数据时，`onReadyStateChange`　回调函数会被调用一次或多次。调用时会带一个描述当前正处于什么状态的对象参数。这个对象包含以下一些属性：

- `ready: boolean`

  当渲染需要的数据子集准备好时，ready为true

- `done: boolean`

  当渲染需要的所有数据准备好时，done为true

- `error: ?Error`

　当有错误发生时，该值为一个 `Error`实例。其它情况下该值为 `null`

- `stale: boolean`

  当强制请求数据时，如果在发起服务端请求之前，客户端已经拥有数据，`ready`已经为true的情况下，该值为true。

- `aborted: boolean`

  当请求被取消时，该值为true。

## 例子

### 从服务器获取数据

如果客户端数据不满足需求导致发起服务端请求获取更多数据，以下状态会发生:

1. 一次`ready`被设置为false。
2. 一次`ready` 和 `done` 设置为true。

### 从客户端恢复数据

如果客户端已经拥有足够的数据，则Relay不需要发送服务端请求，以下状态会发生：

1. 一次 `ready` 和 `done` 设置为true.

### 服务端服务

如果服务器请求结果是加载数据错误，以下状态会发生:

1.一次`ready`设置为false
2.一次`error`设置为`Error`对象

需要注意的是 `ready` 和 `done`　仍然为false。

### 有客户端数据的强制请求

当发生强制请求并且客户端已经拥有足够数据的时候，会发生同 **从服务器获取数据**　一样的行为。而且，如果一次强制请求发生时客户端已经拥有足够的渲染数据，以下状态会发生：

1. 一次 `ready`, `done`, 和 `stale` 设置为 true。
2. 一次 `ready` 和 `done` 设置为true, 但 `stale` 设置为false。
