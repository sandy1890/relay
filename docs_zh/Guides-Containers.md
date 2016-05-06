---
id: guides-containers
title: Containers
layout: docs
category: Guides
permalink: docs/guides-containers.html
next: guides-routes
---

首选的数据需求定义方式是通过`Relay.Container` — 允许React组件编写数据需求的高级React组件

类似于React组件的 `render` 方法并不直接修改原生视图，Relay容器也不直接获取数据。取而代之，容器声明一个需要渲染的数据 *说明(specification)*。Relay框架确保这些数据在渲染之前已经可用。

## 一个复杂的例子

作为开始，我们先创建一个简单的react版本组件`<ProfilePicture>`,该组件展示用户个人照片和一个调整照片尺寸的滑块

### 基础React组件
以下是`<ProfilePicture>`组件的基本实现，代码中忽略了样式定义以便我们能更好的关注功能：

```
class ProfilePicture extends React.Component {
  render() {
    // Expects the `user` prop to have the following shape:
    // {
    //   profilePhoto: {
    //     uri,
    //     size
    //   }
    // }
    var user = this.props.user;
    return (
      <View>
        <Image uri={user.profilePhoto.uri} width={...} />
        <Slider onChange={value => this.setSize(value)} />
      </View>
    );
  }

  // Update the size of the photo
  setSize(photoSize) {
    // TODO: Fetch the profile photo URI for the given size...
  }
}
```

### Data Dependencies With GraphQL

在Relay中，数据依赖使用[GraphQL](https://github.com/facebook/graphql)来进行描述。对于`<ProfilePicture>`组件,它的数据依赖可以用下面的方式表示。注意这个描述与组件本身期望的 `user`　属性结构完全匹配。


```
Relay.QL`
  # This fragment only applies to objects of type `User`.
  fragment on User {
    # Set the `size` argument to a GraphQL variable named `$size` so that we can
    # later change its value via the slider.
    profilePhoto(size: $size) {
      # Get the appropriate URI for the given size, for example on a CDN.
      uri,
    },
  }
`
```

### Relay 容器

有了简单的React组件以及GraphQL片断，我们现在　定义一个　`容器（Container）`来告诉Relay组件的数据需求。我们先看一下下面的代码，然后再解释代码的作用：

```
class ProfilePicture extends React.Component {/* as above */}

// Export a *new* React component that wraps the original `<ProfilePicture>`.
module.exports = Relay.createContainer(ProfilePicture, {
  // Specify the initial value of the `$size` variable.
  initialVariables: {
    size: 32
  },
  // For each of the props that depend on server data, we define a corresponding
  // key in `fragments`. Here, the component expects server data to populate the
  // `user` prop, so we'll specify the fragment from above as `fragments.user`.
  fragments: {
    user: () => Relay.QL`
      fragment on User {
        profilePhoto(size: $size) {
          uri,
        },
      }
    `,
  },
});
```

## 容器是高级组件

Relay容器是高级组件 — `Relay.createContainer`是接受React组件作为输入然后输出返回一个新组件作为输出的函数。这意味着容器可以无需干涉内部组件的 `state` 来管理数据和解决程序逻辑。

以下是一个容器渲染时的内部流程：

<div class="diagram">
  <img src="/relay/img/Guides-Containers-HOC-Relay.png" title="Relay Containers" />
</div>

上面的图中:

- 一个父组件会传递一个引用给某个 `User` "数据"。
- 容器 — 为便于调试我们命名为`Relay(ProfilePicture)` —　会为每一个GraphQL片断从本地存储中取回返回信息。
- 容器解析每一个片断（依据其它属性）的结果给`<ProfilePicture>`组件。
- `<ProfilePicture>` 组件接收到常规的JavaScript对像- objects, arrays, strings - 作为 `user` 属性(prop)，然后像普通React组件一样进行渲染。

## 请求不同的数据

上面的例子中留下了一个问题 — 实现 `setSize()`,这个方法将在滑块值变动时修改照片的尺寸。当解析每一个查询结果到组件中时，Relay同时提供了一个 `relay` 属性,这个属性包含了Relay定义的一些方法和其它元数据。这些方法和数据包含了 `variables` — 用于获取当前 `props`的变量 — 和 `setVariables()` — 可以用作根据不同变量值请求不同数据的回调函数

```{5-6,11,26-28}
class ProfilePicture extends React.Component {
  render() {
    // Access the resolved data for the `user` fragment.
    var user = this.props.user;
    // Access the current `variables` that were used to fetch the `user`.
    var variables = this.props.relay.variables;
    return (
      <View>
        <Image
          uri={user.profilePhoto.uri}
          width={variables.size}
        />
        <Slider onChange={value => this.setSize(value)} />
      </View>
    );
  }

  // Update the size of the photo.
  setSize(photoSize) {
    // `setVariables()` tells Relay that the component's data requirements have
    // changed. The value of `props.relay.variables` and `props.user` will
    // continue to reflect their previous values until the data for the new
    // variables has been fetched from the server. As soon as data for the new
    // variables becomes available, the component will re-render with an updated
    // `user` prop and `variables.size`.
    this.props.relay.setVariables({
      size: photoSize,
    });
  }
}
```

## 容器构建

React和Relay支持通过　*composition*　创建任意的复杂应用。大型的组件可以通过组合小的组件创建，帮助我们创建模块化，健壮的应用。在Relay中构建容器包含两个方面：
- 构建视图逻辑和
- 构建数据描述

让我们探索一下如何通过`<Profile>`组件构建前面提到的　`<ProfilePicture>` 容器

### 构建视图 - 普通React

视图构件完全与通常对于React的认知一致 - Relay容器本身就是标准的React组件.下面是 `<Profile>`组件:

```{8-9}
class Profile extends React.Component {
  render() {
    // Expects a `user` with a string `name`, as well as the information
    // for `<ProfilePicture>` (we'll get that next).
    var user = this.props.user;
    return (
      <View>
        {/* It works just like a React component, because it is one! */}
        <ProfilePicture user={user} />
        <Text>{user.name}</Text>
      </View>
    );
  }
}
```

### 构建片断

片断构件工作方式相当于 — 一个父容器的片断为其下的每一个子组件构建片断。这样， `<Profile>`组件需要的 `User` 信息就需要通过`<ProfilePicture>`容器来获取。

Relay容器提供了一个静态的 `getFragment()` 方法来返回组件需要的片断引用:

```{15}
class Profile extends React.Component {/* as above */}

module.exports = Relay.createContainer(Profile, {
  fragments: {
    // This `user` fragment name corresponds to the prop named `user` that is
    // expected to be populated with server data by the `<Profile>` component.
    user: () => Relay.QL`
      fragment on User {
        # Specify any fields required by `<Profile>` itself.
        name,

        # Include a reference to the fragment from the child component. Here,
        # the `user` is the name of the fragment specified on the child
        # `<ProfilePicture>`'s `fragments` definition.
        ${ProfilePicture.getFragment('user')},
      }
    `,
  }
});
```
最终的数据声明对应如下的GraphQL:

```
`
  fragment Profile on User {
    name,
    ...ProfilePhoto,
  }

  fragment ProfilePhoto on User {
    profilePhoto(size: $size) {
      uri,
    },
  }
`
```
需要注意的是，当构建fragments时，fragment的类型必须与父容器中定义的字段一直。比如，如果将`Story`类型嵌入到父容器的 `User` 字段将没有任何意义。如果发生这样的情形，Relay和GraphQL会提供有用的错误信息

## 渲染容器

目前我们已经学习到，Relay容器声明数据作为GraphQL片断。这就是说，例如　`<ProfilePicture>` 容器不仅可以被`<Profile>`组件也可以被其它任何获取`User`的容器嵌入。

我们甚至准备让Relay为所有组件填充所有数据需求并渲染它们。但是，这样会有一个问题。为了精确的使用GraphQL获取数据，我们需要一个查询的根节点。比如，我们需要将 `<Profile>` 组件的fragment组织到一个具体的　`User` 类型节点。

在Relay框架中，查询根节点被通过 **Route**　进行定义。继续学习关于Relay路由（routes）的知识。
