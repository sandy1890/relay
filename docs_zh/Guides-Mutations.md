---
id: guides-mutations
title: Mutations
layout: docs
category: Guides
permalink: docs/guides-mutations.html
next: guides-network-layer
---

到目前为卡，我们只介绍了如使使用GraphQL终端去处理数据获取的查询部分。这篇指南里，你将学会如何使用Relay进行mutations(变化) – 由数据字段变化引起的一组写入至存储区的操作

## 完整的例子

在突然转向mutations API之前，让我们先来看一个完整的例子。下面，我们从`Relay.Mutation`　扩展一个子类来创建一个可以像stroy一样使用的mutation。

```
class LikeStoryMutation extends Relay.Mutation {
  // This method should return a GraphQL operation that represents
  // the mutation to be performed. This presumes that the server
  // implements a mutation type named ‘likeStory’.
  getMutation() {
    return Relay.QL`mutation {likeStory}`;
  }
  // Use this method to prepare the variables that will be used as
  // input to the mutation. Our ‘likeStory’ mutation takes exactly
  // one variable as input – the ID of the story to like.
  getVariables() {
    return {storyID: this.props.story.id};
  }
  // Use this method to design a ‘fat query’ – one that represents every
  // field in your data model that could change as a result of this mutation.
  // Liking a story could affect the likers count, the sentence that
  // summarizes who has liked a story, and the fact that the viewer likes the
  // story or not. Relay will intersect this query with a ‘tracked query’
  // that represents the data that your application actually uses, and
  // instruct the server to include only those fields in its response.
  getFatQuery() {
    return Relay.QL`
      fragment on LikeStoryPayload {
        story {
          likers {
            count,
          },
          likeSentence,
          viewerDoesLike,
        },
      }
    `;
  }
  // These configurations advise Relay on how to handle the LikeStoryPayload
  // returned by the server. Here, we tell Relay to use the payload to
  // change the fields of a record it already has in the store. The
  // key-value pairs of ‘fieldIDs’ associate field names in the payload
  // with the ID of the record that we want updated.
  getConfigs() {
    return [{
      type: 'FIELDS_CHANGE',
      fieldIDs: {
        story: this.props.story.id,
      },
    }];
  }
  // This mutation has a hard dependency on the story's ID. We specify this
  // dependency declaratively here as a GraphQL query fragment. Relay will
  // use this fragment to ensure that the story's ID is available wherever
  // this mutation is used.
  static fragments = {
    story: () => Relay.QL`
      fragment on Story {
        id,
      }
    `,
  };
}
```

以下是一个在 `LikeButton` 组件上使用这个 mutation的例子：

```
class LikeButton extends React.Component {
  _handleLike = () => {
    // To perform a mutation, pass an instance of one to `Relay.Store.commitUpdate`
    Relay.Store.commitUpdate(new LikeStoryMutation({story: this.props.story}));
  }
  render() {
    return (
      <div>
        {this.props.story.viewerDoesLike
          ? 'You like this'
          : <button onClick={this._handleLike}>Like this</button>
        }
      </div>
    );
  }
}

module.exports = Relay.createContainer(LikeButton, {
  fragments: {
    // You can compose a mutation's query fragments like you would those
    // of any other RelayContainer. This ensures that the data depended
    // upon by the mutation will be fetched and ready for use.
    story: () => Relay.QL`
      fragment on Story {
        viewerDoesLike,
        ${LikeStoryMutation.getFragment('story')},
      }
    `,
  },
});
```

在这个详细的例子里，`LikeButton` 组件唯一关注的字段是　`viewerDoesLike`。Relay会比较高级查询`LikeStoryMutation`后决定哪些字段会包含在服务器返回值中成为mutation返回值。这个字段值来自于这个返回值。应用里其它地方的组件或许也对点赞数或占赞内容感兴趣。因为这些字段会自动添加到Relay的追踪查询里，`LikeButton`并不需要明确的要求它们。

## Mutation属性

所有我们在构建时传递给mutation的属性都可以通过实例的`this.props`变为可用的方法。像组件里使用Relay容器一样，对于匹配的已经定义fragment的属性将被　Relay使用查询数据填充。

```
class LikeStoryMutation extends Relay.Mutation {
  static fragments = {
    story: () => Relay.QL`
      fragment on Story {
        id,
        viewerDoesLike,
      }
    `,
  };
  getMutation() {
    // Here, viewerDoesLike is guaranteed to be available.
    // We can use it to make this mutation polymorphic.
    return this.props.story.viewerDoesLike
      ? Relay.QL`mutation {unlikeStory}`
      : Relay.QL`mutation {likeStory}`;
  }
  /* ... */
}
```

## Fragment变量

像在容器里一样[Relay容器](guides-containers.html),我们可以基于之前的变量和运行时环境，通过mutaions的fragment构建器准备一些可用的变量。　

```
class RentMovieMutation extends Relay.Mutation {
  static initialVariables = {
    format: 'hd',
    lang: 'en-CA',
  };
  static prepareVariables = (prevVariables) => {
    var overrideVariables = {};
    if (navigator.language) {
      overrideVariables.lang = navigator.language;
    }
    var formatPreference = localStorage.getItem('formatPreference');
    if (formatPreference) {
      overrideVariables.format = formatPreference;
    }
    return {...prevVariables, ...overrideVariables};
  };
  static fragments = {
    // Now we can use the variables we've prepared to fetch movies
    // appropriate for the viewer's locale and preferences
    movie: () => Relay.QL`
      fragment on Movie {
        posterImage(lang: $lang) { url },
        trailerVideo(format: $format, lang: $lang) { url },
      }
    `,
  };
}
```

## 高级查询

在系统里修改某个东西，会引发其它部分按顺序变化的连锁效应。试想一个我们用来接受朋友添加请求的mutaion。这会产生巨大的影响：

- 每个人的朋友总数都会增加
- 一方面访问者的朋友列表会增加一个新朋友
- 另一方面访问者本身也会被加入到被访问者的朋友列表
- 访问者的朋友关系状态会变化

设计一个高级查询来包含所有可能变化的字段：

```
class AcceptFriendRequestMutation extends Relay.Mutation {
  getFatQuery() {
    // This presumes that the server-side implementation of this mutation
    // returns a payload of type `AcceptFriendRequestPayload` that exposes
    // `friendEdge`, `friendRequester`, and `viewer` fields.
    return Relay.QL`
      fragment on AcceptFriendRequestPayload {
        friendEdge,
        friendRequester {
          friends,
          friendshipStatusWithViewer,
        },
        viewer {
          friends,
        },
      }
    `;
  }
}
```

除了一个重要的区别，高级查询看起来和其它GraphQL查询一样。我们知道有些字段是混合型的，（比如 `friendEdge` 和 `friends`) ，但我们并没有通过任何子查询来命名它们的子元素。在这里，我们实际上是告诉Relay，所有这些混合型的字段会作为当前mutations的结果发生变化。

> 注意
>
> 当设计一个高级查询的时候, 考虑所有可能因为mutation而发生变化的数据 – 而不仅仅是你的应用当前正在使用的数据。我不需要担心获取过多数据；如果我们的应用没有通过‘跟踪查询（tracked query）’在实际需要的时候请求过它，这些查询并不会被执行。如果我们在高级查询中省略了这些字段，将来，当我们用新的数据依赖添加新视图或者为已经存在的视图增加新的数据依赖时，我们可能会发现到一些数据错误。

## Mutator配置

我们需要向Relay说明如何使用从每一个mutaion响应数据去更新客户端的数据存储。我们可能通过使用以下的一个或多个mutation类型来配置mutaion实现：

### `FIELDS_CHANGE`

返回数据的所有字段都将通过DataID关联客户端store中的数据，使用新的返回值同客户端数据进行合并。

#### 参数

- `fieldIDs: {[fieldName: string]: DataID | Array<DataID>}`

  返回数据中的fieldName和store中一个或多个DataIDs进行映射的map

#### 示例

```
class RenameDocumentMutation extends Relay.Mutation {
  // This mutation declares a dependency on a document's ID
  static fragments = {
    document: () => Relay.QL`fragment on Document { id }`,
  };
  // We know that only the document's name can change as a result
  // of this mutation, and specify it here in the fat query.
  getFatQuery() {
    return Relay.QL`
      fragment on RenameDocumentMutationPayload { updatedDocument { name } }
    `;
  }
  getVariables() {
    return {id: this.props.document.id, newName: this.props.newName};
  }
  getConfigs() {
    return [{
      type: 'FIELDS_CHANGE',
      // Correlate the `updatedDocument` field in the response
      // with the DataID of the record we would like updated.
      fieldIDs: {updatedDocument: this.props.document.id},
    }];
  }
  /* ... */
}
```

### `NODE_DELETE`

在返回数据中指定一个上级，一个连接点和一个或我个DataIDs,Relay会从连接点里删除指定的节点并从store中删除关联的数据。

#### 参数

- `parentName: string`

  返回值中表示连接点上级的字段名

- `parentID: string`

  包含连接点的上级节点的DataID

- `connectionName: string`

  返回数据中表示连接点的字段名

- `deletedIDFieldName: string`

  返回数据中包含需要被删除的节点DataID的字段

#### 示例

```
class DestroyShipMutation extends Relay.Mutation {
  // This mutation declares a dependency on an enemy ship's ID
  // and the ID of the faction that ship belongs to.
  static fragments = {
    ship: () => Relay.QL`fragment on Ship { id, faction { id } }`,
  };
  // Destroying a ship will remove it from a faction's fleet, so we
  // specify the faction's ships connection as part of the fat query.
  getFatQuery() {
    return Relay.QL`
      fragment on DestroyShipMutationPayload {
        destroyedShipID,
        faction { ships },
      }
    `;
  }
  getConfigs() {
    return [{
      type: 'NODE_DELETE',
      parentName: 'faction',
      parentID: this.props.ship.faction.id,
      connectionName: 'ships',
      deletedIDFieldName: 'destroyedShipID',
    }];
  }
  /* ... */
}
```

### `RANGE_ADD`

在返回数据中指定一个上级，一个连接点和新创建的修整数据（edge)。　Rellay将把这些节点添加到store中并且根据rangeBehavior指定的规则将节点附加到连接点上。

#### 参数

- `parentName: string`

  返回值中表示连接点上级的字段名

- `parentID: string`

  包含连接点的上级节点的DataID

- `connectionName: string`

  返回数据中表示连接点的字段名

- `edgeName: string`

  返回数据中表示最新创建的修整数据（edge）的字段名

- `rangeBehaviors: {[call: string]: GraphQLMutatorConstants.RANGE_OPERATIONS} | (connectionArgs: {[argName: string]: string}) => $Enum<GraphQLMutatorConstants.RANGE_OPERATIONS>`

  A map between printed, dot-separated GraphQL calls *in alphabetical order* and the behavior we want Relay to exhibit when adding the new edge to connections under the influence of those calls or a function accepting an array of connection arguments, returning that behavior.

比如, `rangeBehaviors` 可以用以下这种方式编写:

```
const rangeBehaviors = {
  // When the ships connection is not under the influence
  // of any call, append the ship to the end of the connection
  '': 'append',
  // Prepend the ship, wherever the connection is sorted by age
  'orderby(newest)': 'prepend',
};
```

或者用以下的方法编写和上面同样的规则：

```
const rangeBehaviors = ({orderby}) => {
  if (orderby === 'newest') {
    return 'prepend';
  } else {
    return 'append';
  }
};

```

Behaviors可以是 `'append'`, `'ignore'`, `'prepend'`, `'refetch'`, 或 `'remove'`　其中一个值。

#### 示例

```
class IntroduceShipMutation extends Relay.Mutation {
  // This mutation declares a dependency on the faction
  // into which this ship is to be introduced.
  static fragments = {
    faction: () => Relay.QL`fragment on Faction { id }`,
  };
  // Introducing a ship will add it to a faction's fleet, so we
  // specify the faction's ships connection as part of the fat query.
  getFatQuery() {
    return Relay.QL`
      fragment on IntroduceShipPayload {
        faction { ships },
        newShipEdge,
      }
    `;
  }
  getConfigs() {
    return [{
      type: 'RANGE_ADD',
      parentName: 'faction',
      parentID: this.props.faction.id,
      connectionName: 'ships',
      edgeName: 'newShipEdge',
      rangeBehaviors: {
        // When the ships connection is not under the influence
        // of any call, append the ship to the end of the connection
        '': 'append',
        // Prepend the ship, wherever the connection is sorted by age
        'orderby(newest)': 'prepend',
      },
    }];
  }
  /* ... */
}
```

### `RANGE_DELETE`

在返回数据中指定一个上级, 一个连接点, 一个或多个DataIDs和在上级与连接点之间的路径，Relay会从连接点中删除节点但是保留store中关联的数据

#### 参数

- `parentName: string`

  返回值中表示连接点上级的字段名

- `parentID: string`

  包含连接点的上级节点的DataID

- `connectionName: string`

  返回数据中表示连接点的字段名

- `deletedIDFieldName: string | Array<string>`

　返回数据中包含将要移除节点的DataID或是将从连接点中删除的节点路径

- `pathToConnection: Array<string>`

  包含上级和连接点之间字段名的数组，含上级和连接点

#### 示例

```
class RemoveTagMutation extends Relay.Mutation {
  // This mutation declares a dependency on the
  // todo from which this tag is being removed.
  static fragments = {
    todo: () => Relay.QL`fragment on Todo { id }`,
  };
  // Removing a tag from a todo will affect its tags connection
  // so we specify it here as part of the fat query.
  getFatQuery() {
    return Relay.QL`
      fragment on RemoveTagMutationPayload {
        todo { tags },
        removedTagIDs,
      }
    `;
  }
  getConfigs() {
    return [{
      type: 'RANGE_DELETE',
      parentName: 'todo',
      parentID: this.props.todo.id,
      connectionName: 'tags',
      deletedIDFieldName: 'removedTagIDs',
    }];
  }
  /* ... */
}
```

### `REQUIRED_CHILDREN`

`REQUIRED_CHILDREN` 配置用于附加额外的下级到mutation查询。 比如，用于当通过mutation创建新对象的时候获取字段（因为在这之初胼没有为这个对象获取任何数据，Relay通常不会尝试获取字段）。

`REQUIRED_CHILDREN`　获取的数据不会写入到客户端store，但是你可以在使用`commitUpdate()`时在`onSuccess`回调中添加代码处理这些数据:

```
Relay.Store.commitUpdate(
  new CreateCouponMutation(),
  {
    onSuccess: response => this.setState({
      couponCount: response.coupons.length,
    }),
  }
);
```

#### 参数

- `children: Array<RelayQuery.Node>`

#### 示例

```
class CreateCouponMutation extends Relay.Mutation<Props> {
  getMutation() {
    return Relay.QL`mutation {
      create_coupon(data: $input)
    }`;
  }

  getFatQuery() {
    return Relay.QL`
      // Note the use of `pattern: true` here to show that this
      // connection field is to be used for pattern-matching only
      // (to determine what to fetch) and that Relay shouldn't
      // require the usual connection arguments like (`first` etc)
      // to be present.
      fragment on CouponCreatePayload @relay(pattern: true) {
        coupons
      }
    `;
  }

  getConfigs() {
    return [{
      // If we haven't shown the coupons in the UI at the time the
      // mutation runs, they've never been fetched and the `coupons`
      // field in the fat query would normally be ignored.
      // `REQUIRED_CHILDREN` forces it to be retrieved anyway.
      type: RelayMutationType.REQUIRED_CHILDREN,
      children: [
        Relay.QL`
          fragment on CouponCreatePayload {
            coupons
          }
        `,
      ],
    }];
  }
}
```

## 乐观更新

所有我们目前展示的mutaion在更新客户端store之前都需要等待服务端返回。Relay提供给我们一种机制，可以在获取服务器返回之前乐观的返回一个我们期待在mutation成功执行后服务才会返回的数据。

下面让我们来为 `LikeStoryMutation` 创建一个乐观更新的例子：

```
class LikeStoryMutation extends Relay.Mutation {
  /* ... */
  // Here's the fat query from before
  getFatQuery() {
    return Relay.QL`
      fragment on LikeStoryPayload {
        story {
          likers {
            count,
          },
          likeSentence,
          viewerDoesLike,
        },
      }
    `;
  }
  // Let's craft an optimistic response that mimics the shape of the
  // LikeStoryPayload, as well as the values we expect to receive.
  getOptimisticResponse() {
    return {
      story: {
        id: this.props.story.id,
        likers: {
          count: this.props.story.likers.count + (this.props.story.viewerDoesLike ? -1 : 1),
        },
        viewerDoesLike: !this.props.story.viewerDoesLike,
      },
    };
  }
  // To be able to increment the likers count, and flip the viewerDoesLike
  // bit, we need to ensure that those pieces of data will be available to
  // this mutation, in addition to the ID of the story.
  static fragments = {
    story: () => Relay.QL`
      fragment on Story {
        id,
        likers { count },
        viewerDoesLike,
      }
    `,
  };
  /* ... */
}
```

你并不需要模拟所有的返回值。这个示例中，我们仅处理了点赞语句。当服务端返回值时，Relay会使用真实的返回值作为实际的数据来源，在这之前，乐观的返回值会被正确处理，允许使用我们产品的人在执行操作后获得更好的即时反馈。
