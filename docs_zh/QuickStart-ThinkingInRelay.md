---
id: thinking-in-relay
title: Thinking In Relay
layout: docs
category: Quick Start
permalink: docs/thinking-in-relay.html
next: videos
---

Relay的数据获取方式深受React的影响。实际上，React将复杂的界面分拆为可重用的 **组件(components)**,使开发者可以分别创建独立的应用功能块，减少应用各独立部分的耦合。更重要的是这些组件是 **声明式（declarative）**　的：允许开发者定义对于给定状态的组件UI样式，并且不用担心如何将数据渲染至UI. 不同于以往直接操作原生视图的(例如 DOM)方式，React使用一个UI描述自动处理必要的命令。

下面用一些实际的用例来帮助理解我们如何融合这种思想到Relay。继续下面的内容前，假设你对于React已经有一个基本的理解。

## 为视图获取数据

根据经验，绝大多数产品需要定义一种行为: 首先显示一个加载指示器，为视图层获取所有数据，等数据加载完后，渲染所有的视图。


一种解决方案是创建一个根组件，获取其下的所有子视图需要的数据.这种方式会产生严重的耦合：每一个组件的变化都需要更改与它相关的所有根组件以及其它相关组件，这意味着产生bugs的可能增加和降低开发进度。这种方案最终并没有发挥React组件模型的优势。更合理的定义数据依赖的地方是在 **组件（components）** 内部。


另外一个合理的解决方案是使用 `render()` 作为获取初始化数据的时机。我们可以先简单的渲染应用程序，然后看组件需要什么数据，获取并重新渲染。听起来很不错，但是问题在于 *组件需要使用数据来决定渲染！*　也就是说，这将导致分级式的数据获取: 首先渲染根组件并且找到根组件需要的数据，然后渲染子组件获取子组件需要的数据，至到整个组件树的底端。如果每一级都需要发起网络请求，整个渲染过程将会变得很慢，需要多次数据交互。我们需要一种能在所有步骤之前就能决定所需数据或是*静态的*方式。


我们最终选择了静态方法：组件可以返回一个独立于视图树的查询树(query-tree),用于描述组件的数据依赖。Relay可以利用这个查询树获取单一步骤所需要的所有数据并渲染组件。这种方式的问题在于需要找到一个合适的机制来描述查询树和一种高效的从服务器获取数据的方式（例如：只通过一次网络请求获取）。这对于GraphQL是最合适的应用场景，因为它提供了一种 *描述数据依赖作为数据* 的语法，不需要调用任何实际的API.注意Promises和Observables虽然通常作为可选项，但是他们提供了为 *透明的命令* 和类似批量查询这样的许多优化。

## 数据组件(Components)也可以称为容器(Containers)


Relay允许开发者通过创建 **容器(containers)** 将React组件声明为带有数据依赖的组件。它们也是符合React定义的常规组件。一个关键的设计约束是React组件的可重用性，所以Relay容器也必须满足可重用性。举个例子,有一个 `<Story>`组件实现了对所有`Story`条目进行渲染的视图。某个具体的story将根据传入这个组件的数据被渲染: `<Story story={...} />`。这在GraphQL里叫做 **fragments**:定义了对给定类型对像所需数据的命名查询片断。我们可以像下面这样来描述 `<Story>`组件需要的数据:


```
fragment on Story {
  text,
  author {
    name,
    photo
  }
}
```

接下来，我们可以使用这个fragment去定义Story容器：

```javascript
// Plain React component.
// Usage: `<Story story={ ... } />`
class Story extends React.Component { ... }

// "Higher-order" component that wraps `<Story>`
var StoryContainer = Relay.createContainer(Story, {
  fragments: {
    // Define a fragment with a name matching the `story` prop expected above
    story: () => Relay.QL`
      fragment on Story {
        text,
        author { ... }
      }
    `
  }
})
```

## 渲染

在React里，渲染一个视图需要两个输入值：需要渲染的组件和容器节点。渲染Relay容器也类似：需要一个被渲染的 *container(容器)*和一个开始查询的根视图。我们必须确定容器的查询是可执行的并且希望在获取数据时展示一个加载指示器。和`ReactDOM.render(component, domNode)`类似，Relay提供`<RelayRootContainer Component={...} route={...}>` 来实现。Component是需要被渲染的组件，route提供了定义获取指定数据的查询。以下代码展示了如何渲染`<StoryContainer>`:

```javascript
ReactDOM.render(
  <RelayRootContainer
    Component={StoryContainer}
    route={{
      queries: {
        story: () => Relay.QL`
          query {
            node(id: "123") /* our `Story` fragment will be added here */
          }
        `
      },
    }}
  />,
  rootEl
)
```

`RelayRootContainer` 会处理查询获取的数据；同缓存数据对比，获取缺少的信息，更新缓存，最后当所有数据准备好之后渲染`StoryContainer`。默认情况下，当正在获取数据时，将什么也不会渲染，但通过 `readerLoading`属性可以定义正在加载视图。就像React允许开发者不需要操作底层视图渲染视图一样，Relay 和 `RelayRootContainer`也可以不通过网络获取数据而直接渲染视图。

## 数据隐藏

我们发现，使用常规的方式获取数据，两个组件之间通常会产生 *不明确的依赖*。比如`<StoryHeader>`或许使用了某些没有确定已经获取的数据。这些数据通常在系统的其它部分中获取,如 `<Story>`。当我们修改 `<Story>` 并且移除了数据获取逻辑的时候， `<StoryHeader>`组件将突然莫名的失效。这类bugs通常不会立即被发现，特别是在大型团队开发的大型应用里。手动和自行化测试也帮助有限:可以确定的是这种类型的系统问题最好通过框架解决。


我们已经看到Realy容器（containers）确保证了在组件渲染之前GraphQL fragments已经获取了数据。但是容器（containers）也提供了另外一种非即时的方式：**data masking**。Relay只允许组件访问组件自己在`fragments`中指定的数据。所以，如果一个组件查询Story的 `text`字段，另外一个查询 `author`字段，每一个组件只能访问到它自己请求的字段。实际上，组件甚至不能访问它的子组件请求的数据：访问子组件的数据同样会破坏封装

Relay考虑得更远：它在组件的`props`上使用隐含的标识来验证是否在渲染一个组件时已经明确的获取了需要的数据。如果`<Story>`渲染`<StoryHeader>`但是忘记包含了fragment,Relay会发出`<StoryHeader>`缺少数据的警告。实际上，甚至当其它组件获取`<StoryHeader>`需要的数据时，Relay也会发出警告。这类警告会提示我们，虽然现在一切正常，但将来有很高的可能性会导致程序破坏。


# 结论

GraphQL提供了一个创建高效，解耦客户端应用的强大的工具。Relay基于这些功能提供了一个 **声明式的数据获取** 框架。通过分离如何获取数据和需要获取什么数据，Relay帮助开发者创建健壮、清晰和高性能的应用。这是一个对react提出的组件为中心的思想的补充。当这一项技术 — React, Relay, 和 GraphQL — 都足够强大的时候，这几项技术的组合会是一个允许我们　**更快速开发**　和　**提供高品质大规模app**　的　**UI平台**。
