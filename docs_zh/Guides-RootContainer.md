---
id: guides-root-container
title: Root Container
layout: docs
category: Guides
permalink: docs/guides-root-container.html
next: guides-ready-state
---

现在，我们已经了解了实现数据定义的两个工具：

 - **Relay.Route** 用于定义查询根节点。
 - **Relay.Container** 用于组件中的片断定义。

使用这两个工具构建可发送到服务器获取数据的完整GraphQL查询,我们还需要使用 **Relay.RootContainer**。

## 组件和路由

**Relay.RootContainer** 是一个包含 `Component` 和 `route` 属性的React组件, 处理渲染组件`Component`需要的数据。

```
ReactDOM.render(
  <Relay.RootContainer
    Component={ProfilePicture}
    route={profileRoute}
  />,
  container
);
```

当上面的　**Relay.RootContainer**　组件渲染完成之后，Relay会构造一个查询并发送到服务器。当从服务器取得所有需要的数据后就会渲染`ProfilePicture`组件。带有fragments的属性(props)会包含从服务器获得的数据。

`组件（Component）` 和 `路由（route）`一旦发生变化，**Relay.RootContainer** 会立即开始处理新的数据请求。

## 渲染回调整

**Relay.RootContainer** 接受三个可选的回调属性，提供给我们更细粒度的控制渲染行为的方式。

### `renderLoading`

**Relay.RootContainer** 还未获取渲染组件所需的数据时，会处理加载状态的视图。通常在初始化时发生，也可能在组件和路由发生变化时发生。

通常在渲染初始化时，不会输出任何视图。如果我们预加载了一个视图，当渲染完成时，会渲染新的视图。

我们可以通过设置`renderLoading`属性来实现预加载视图：

We can change this behavior by supplying the `renderLoading` prop:

```{4-6}
<Relay.RootContainer
  Component={ProfilePicture}
  route={profileRoute}
  renderLoading={function() {
    return <div>Loading...</div>;
  }}
/>
```
上面这个代码片断设置　**Relay.RootContainer** 在进行数据加载的时候输出一个 "Loading..." 文本。

`renderLoading` 回调函数可以模拟返回值为｀未定义(undefined)｀　时的默认的行为。需要注意的是这和`renderLoading`返回 null值的情形不一样，当返回null时，无论在数据加载时还是之前已经有过视图输出过，都不会有任何视图输出。

### `renderFetched`

当所有渲染视图必须的数据都准备好时，**Relay.RootContainer**　会默认输出关联的　｀组件（Component）｀。我们也可以通过设置　`renderFetched` 属性来改变这种默认行为:

```{4-10}
<Relay.RootContainer
  Component={ProfilePicture}
  route={profileRoute}
  renderFetched={function(data) {
    return (
      <ScrollView>
        <ProfilePicture {...data} />
      </ScrollView>
    );
  }}
/>
```

上面这个代码片断设置 **Relay.RootContainer** 在所需要数据准备好后，输出一个带有 `ScrollView` 组件的　`ProfilePicture` 组件。

`renderFetched` 回调函数被调用时带有一个 `data` 参数, 这个参数是一个可以通过属性名为键名查询对应数据的对像。 这样我们就可以在回调函数中使用这些数据来展示需要的组件。 (比如使用[JSX 延展属性](https://facebook.github.io/react/docs/jsx-spread.html))。

> 注意
>
> 虽然我们可以在 `renderFetched`　回调中访问 `data`对像，但真实的数据是特意处理为不透明的。这防止了 `renderFetched` 从`Component`声明的片断(fragments)创建隐含的依赖。

### `renderFailure`

为了预防　**Relay.RootContainer**　获取｀Component｀所需数据发生错误导致没有输出。可以通过设置`renderFailure` 回调属性实现错误处理：

```{4-11}
<Relay.RootContainer
  Component={ProfilePicture}
  route={profileRoute}
  renderFailure={function(error, retry) {
    return (
      <div>
        <p>{error.message}</p>
        <p><button onClick={retry}>Retry?</button></p>
      </div>
    );
  }}
/>
```

`renderFailure` 回调函数有两个参数: 一个 `Error` 对像和一个重试请求函数。如果错误是服务器端返回的通讯错误，相关信息可以通过 `error.source`获得。

## 强制刷新　Force Fetching

像其它 Relay API一样, **Relay.RootContainer**在向服务器发送请求获取数据之前，会先尝试通过本地客户端存储来获得数据。如果我们希望跳过本地存储信息，直接从服务器获取最新的数据，可以通过设置 `forceFetch`属性来实现。

```{4}
<Relay.RootContainer
  Component={ProfilePicture}
  route={profileRoute}
  forceFetch={true}
/>
```

当设置 `forceFetch` 为true时, **Relay.RootContainer** 将总是发起一个请求到服务器。 也就是说，哪怕所有数据已经在客户端存在，`renderFetched`都会在服务器请求完成之前被执行。

```{5-6,9}
<Relay.RootContainer
  Component={ProfilePicture}
  route={profileRoute}
  forceFetch={true}
  renderFetched={function(data, readyState) {
    var isRefreshing = readyState.stale;
    return (
      <ScrollView>
        <Spinner style={{display: isRefreshing ? 'block' : 'none' }}
        <ProfilePicture {...data} />
      </ScrollView>
    );
  }}
/>
```

当`forceFetch`设置为true并且`renderFetched`已经被客户端调用过，`renderFetched`的第二个调用参数是一个带有布尔型｀stale｀属性的对像。如果在强制刷新数据完成之前`renderFetched` 已经被调用过，这个｀stale｀　值会为true。

## 就绪状态变化

**Relay.RootContainer** 支持一个叫做 `onReadyStateChange` 的属性，这个属性使我们可以收到更细粒度的获取数据时的事件

下一篇文章我们将学习 `onReadyStateChange` [Ready State](guides-ready-state.html).
