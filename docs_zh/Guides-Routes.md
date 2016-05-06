---
id: guides-routes
title: Routes
layout: docs
category: Guides
permalink: docs/guides-routes.html
next: guides-root-container
---

路由（Routes）负责定义Relay应用程序的入口。为了理解为什么路由（routes)是必须的，我们首先需要理解GraphQL查询(queries)和片断(fragements)之间的差别。

> 注意
>
> Relay路由（routes）并没有真正的实现任何URL路由定义功能或浏览历史API.将来我们或许会将RelayRoute重命名为RealyQueryRoots或RelayQueryConfig。关于更多为什么Realay不提供URL-routing特性和针对这种解决方案的讨论，请参看[这篇文章](https://medium.com/@cpojer/relay-and-routing-36b5439bad9).


## 查询（Queries） vs. 片断（Fragments）

在GraphQL中， **查询（queries）**　声明存在于根查询中的类型字段。比如，下面的查询可以通过｀id｀字段和指定值 `123` 获取用户的姓名

```
query UserQuery {
  user(id: "123") {
    name,
  },
}
```

另外一方面，GraphQL **片断（fragments）**　声明存在于任意专有类型中的字段。比如，下面的片断获取某个 `User`　类型的头像图片URI

```
fragment UserProfilePhoto on User {
  profilePhoto(size: $size) {
    uri,
  },
}
```

片断（fragments）可以嵌入其它片断（fragments）或查询（queries）。比如，下面的片断可以用于获取用户`123`的头像图片：

```
query UserQuery {
  user(id: "123") {
    ...UserProfilePhoto,
  },
}
```

另外，片断（fragments）也能获取用户 `123`　所有朋友的头像图片:

```
query UserQuery {
  user(id: "123") {
    friends(first: 10) {
      edges {
        node {
          ...UserProfilePhoto,
        },
      },
    },
  },
}
```

因此Realy容器中定义了片断（fragments）而不是查询（queries），这样片断（fragments）就能够轻松的嵌入各种上下文。就像React组件，Relay容器高重用性的。

## 路由（Routes）和查询（Queries）

路由（Routes）是定义了一系列根查询（Queries）和输入参数的对像。以下是一个简单的可以用来渲染用户`123`个人主页的路由：

```
var profileRoute = {
  queries: {
    // Routes declare queries using functions that return a query root. Relay
    // will automatically compose the `user` fragment from the Relay container
    // paired with this route on a Relay.RootContainer
    user: () => Relay.QL`
      # In Relay, the GraphQL query name can be optionally omitted.
      query { user(id: $userID) }
    `,
  },
  params: {
    // This `userID` parameter will populate the `$userID` variable above.
    userID: '123',
  },
  // Routes must also define a string name.
  name: 'ProfileRoute',
};
```

如果我们想要为指定用户创建一个该路由的实例，我们可以从抽象类`Relay.Route`继承一个子类。`Relay.Route` 使定义可重复使用的一系统查询和请求参数变得容易：

```
class ProfileRoute extends Relay.Route {
  static queries = {
    user: () => Relay.QL`
      query { user(id: $userID) }
    `,
  };
  static paramDefinitions = {
    // By setting `required` to true, `ProfileRoute` will throw if a `userID`
    // is not supplied when instantiated.
    userID: {required: true},
  };
  static routeName = 'ProfileRoute';
}
```
现在我们能初始化一个获取用户`123`数据的　`ProfileRoute`　路由了：

```
// Equivalent to the object literal we created above.
var profileRoute = new ProfileRoute({userID: '123'});
```
到目前为止，我们已经可以创建针对指定用户ID的路由了.例如,我们准备构建一个通过｀userID｀查询参数获取指定用户数据的路由，我们可以像下面这样编写代码：

```
window.addEventListener('popstate', () => {
  var userID = getQueryParamFromURI('userID', document.location.href);
  var profileRoute = new ProfileRoute({userID: userID});
  ReactDOM.render(
    <Relay.RootContainer
      Component={UserProfile}
      route={profileRoute}
    />,
    document.getElementById('app')
  );
});
```
