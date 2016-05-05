---
id: tutorial
title: Tutorial
layout: docs
category: Quick Start
permalink: docs/tutorial.html
next: thinking-in-graphql
---

我们将使用 GraphQL mutations　构建一个从九宫格寻找宝藏的游戏。 游戏玩家有三次机会从九宫格里猜出宝藏。通过这个教程，我们能够从头到尾完整的认识一下Relay　--　从服务端的 GraphQL schema 到客户端React应用.

## 准备

我们将使用[Relay Starter Kit](https://github.com/relayjs/relay-starter-kit)开发框架为基础编写这个游戏

```
git clone https://github.com/relayjs/relay-starter-kit.git relay-treasurehunt
cd relay-treasurehunt
npm install
```

## 简单数据库

我们需要一个存储宝藏数据的地方，用于存放宝藏隐藏点以及剩余寻宝机会。此教程中我们将这些数据存在内存中。

```
/**
 * ./data/database.js
 */

// Model types
export class Game {}
export class HidingSpot {}

// Mock data
var game = new Game();
game.id = '1';

var hidingSpots = [];
(function() {
  var hidingSpot;
  var indexOfSpotWithTreasure = Math.floor(Math.random() * 9);
  for (var i = 0; i < 9; i++) {
    hidingSpot = new HidingSpot();
    hidingSpot.id = `${i}`;
    hidingSpot.hasTreasure = (i === indexOfSpotWithTreasure);
    hidingSpot.hasBeenChecked = false;
    hidingSpots.push(hidingSpot);
  }
})();

var turnsRemaining = 3;

export function checkHidingSpotForTreasure(id) {
  if (hidingSpots.some(hs => hs.hasTreasure && hs.hasBeenChecked)) {
    return;
  }
  turnsRemaining--;
  var hidingSpot = getHidingSpot(id);
  hidingSpot.hasBeenChecked = true;
};
export function getHidingSpot(id) {
  return hidingSpots.find(hs => hs.id === id)
}
export function getGame() { return game; }
export function getHidingSpots() { return hidingSpots; }
export function getTurnsRemaining() { return turnsRemaining; }
```

目前我们构建的是一组模拟数据。我们也完全可以将这个部分关联到一个真实的数据库里。

## 创建一个方案(schema)

A GraphQL schema describes your data model, and provides a GraphQL server with
an associated set of resolve methods that know how to fetch data. We will use
一个GraphQL方案（schema）是对数据模型的描述,并提供了一个包含一系列如何获取数据的关联方法的GraphQL服务。
我们将使用
[graphql-js](https://github.com/graphql/graphql-js) 和
[graphql-relay-js](https://github.com/graphql/graphql-relay-js)
来创建方案。

打开我们之前获取的基础框架目录，找到方案定义文件./data/schema.js,将里面的import{...}from database部分内容替换为以下代码

```
/**
 * ./data/schema.js
 */

/* ... */

import {
  Game,
  HidingSpot,
  checkHidingSpotForTreasure,
  getGame,
  getHidingSpot,
  getHidingSpots,
  getTurnsRemaining,
} from './database';
```

删除`./data/schema.js`里｀queryType｀以上的部分


Next, let's define a node interface and type. We need only provide a way for Relay to map from an object to the GraphQL type associated with that object, and from a global ID to the object it points to:

```
var {nodeInterface, nodeField} = nodeDefinitions(
  (globalId) => {
    var {type, id} = fromGlobalId(globalId);
    if (type === 'Game') {
      return getGame(id);
    } else if (type === 'HidingSpot') {
      return getHidingSpot(id);
    } else {
      return null;
    }
  },
  (obj) => {
    if (obj instanceof Game) {
      return gameType;
    } else if (obj instanceof HidingSpot)  {
      return hidingSpotType;
    } else {
      return null;
    }
  }
);
```

Next, let's define our game and hiding spot types, and the fields that are available on each.

```
var gameType = new GraphQLObjectType({
  name: 'Game',
  description: 'A treasure search game',
  fields: () => ({
    id: globalIdField('Game'),
    hidingSpots: {
      type: hidingSpotConnection,
      description: 'Places where treasure might be hidden',
      args: connectionArgs,
      resolve: (game, args) => connectionFromArray(getHidingSpots(), args),
    },
    turnsRemaining: {
      type: GraphQLInt,
      description: 'The number of turns a player has left to find the treasure',
      resolve: () => getTurnsRemaining(),
    },
  }),
  interfaces: [nodeInterface],
});

var hidingSpotType = new GraphQLObjectType({
  name: 'HidingSpot',
  description: 'A place where you might find treasure',
  fields: () => ({
    id: globalIdField('HidingSpot'),
    hasBeenChecked: {
      type: GraphQLBoolean,
      description: 'True if this spot has already been checked for treasure',
      resolve: (hidingSpot) => hidingSpot.hasBeenChecked,
    },
    hasTreasure: {
      type: GraphQLBoolean,
      description: 'True if this hiding spot holds treasure',
      resolve: (hidingSpot) => {
        if (hidingSpot.hasBeenChecked) {
          return hidingSpot.hasTreasure;
        } else {
          return null;  // Shh... it's a secret!
        }
      },
    },
  }),
  interfaces: [nodeInterface],
});
```

Since one game can have many hiding spots, we need to create a connection that we can use to link them together.

```
var {connectionType: hidingSpotConnection} =
  connectionDefinitions({name: 'HidingSpot', nodeType: hidingSpotType});
```

Now let's associate these types with the root query type.

```
var queryType = new GraphQLObjectType({
  name: 'Query',
  fields: () => ({
    node: nodeField,
    game: {
      type: gameType,
      resolve: () => getGame(),
    },
  }),
});
```

With the queries out of the way, let's start in on our only mutation: the one that spends a turn by checking a spot for treasure. Here, we define the input to the mutation (the id of a spot to check for treasure) and a list of all of the possible fields that the client might want updates about after the mutation has taken place. Finally, we implement a method that performs the underlying mutation.

```
var CheckHidingSpotForTreasureMutation = mutationWithClientMutationId({
  name: 'CheckHidingSpotForTreasure',
  inputFields: {
    id: { type: new GraphQLNonNull(GraphQLID) },
  },
  outputFields: {
    hidingSpot: {
      type: hidingSpotType,
      resolve: ({localHidingSpotId}) => getHidingSpot(localHidingSpotId),
    },
    game: {
      type: gameType,
      resolve: () => getGame(),
    },
  },
  mutateAndGetPayload: ({id}) => {
    var localHidingSpotId = fromGlobalId(id).id;
    checkHidingSpotForTreasure(localHidingSpotId);
    return {localHidingSpotId};
  },
});
```

Let's associate the mutation we just created with the root mutation type:

```
var mutationType = new GraphQLObjectType({
  name: 'Mutation',
  fields: () => ({
    checkHidingSpotForTreasure: CheckHidingSpotForTreasureMutation,
  }),
});
```

Finally, we construct our schema (whose starting query type is the query type we defined above) and export it.

```
export var Schema = new GraphQLSchema({
  query: queryType,
  mutation: mutationType
});
```

## Processing the schema

Before going any further, we need to serialize our executable schema to JSON for use by the Relay.QL transpiler, then start up the server. From the command line:

```
npm run update-schema
npm start
```

## Writing the game

Let's tweak the file `./js/routes/AppHomeRoute.js` to anchor our game to the `game` root field of the schema:

```
export default class extends Relay.Route {
  static path = '/';
  static queries = {
    game: () => Relay.QL`query { game }`,
  };
  static routeName = 'AppHomeRoute';
}
```

Next, let's create a file in `./js/mutations/CheckHidingSpotForTreasureMutation.js` and create subclass of `Relay.Mutation` called `CheckHidingSpotForTreasureMutation` to hold our mutation implementation:

```
import Relay from 'react-relay';

export default class CheckHidingSpotForTreasureMutation extends Relay.Mutation {
  static fragments = {
    game: () => Relay.QL`
      fragment on Game {
        id,
        turnsRemaining,
      }
    `,
    hidingSpot: () => Relay.QL`
      fragment on HidingSpot {
        id,
      }
    `,
  };
  getMutation() {
    return Relay.QL`mutation{checkHidingSpotForTreasure}`;
  }
  getCollisionKey() {
    return `check_${this.props.game.id}`;
  }
  getFatQuery() {
    return Relay.QL`
      fragment on CheckHidingSpotForTreasurePayload {
        hidingSpot {
          hasBeenChecked,
          hasTreasure,
        },
        game {
          turnsRemaining,
        },
      }
    `;
  }
  getConfigs() {
    return [{
      type: 'FIELDS_CHANGE',
      fieldIDs: {
        hidingSpot: this.props.hidingSpot.id,
        game: this.props.game.id,
      },
    }];
  }
  getVariables() {
    return {
      id: this.props.hidingSpot.id,
    };
  }
  getOptimisticResponse() {
    return {
      game: {
        turnsRemaining: this.props.game.turnsRemaining - 1,
      },
      hidingSpot: {
        id: this.props.hidingSpot.id,
        hasBeenChecked: true,
      },
    };
  }
}
```

Finally, let's tie it all together in `./js/components/App.js`:

```
import CheckHidingSpotForTreasureMutation from '../mutations/CheckHidingSpotForTreasureMutation';

class App extends React.Component {
  _getHidingSpotStyle(hidingSpot) {
    var color;
    if (this.props.relay.hasOptimisticUpdate(hidingSpot)) {
      color = 'lightGrey';
    } else if (hidingSpot.hasBeenChecked) {
      if (hidingSpot.hasTreasure) {
        color = 'blue';
      } else {
        color = 'red';
      }
    } else {
      color = 'black';
    }
    return {
      backgroundColor: color,
      cursor: this._isGameOver() ? null : 'pointer',
      display: 'inline-block',
      height: 100,
      marginRight: 10,
      width: 100,
    };
  }
  _handleHidingSpotClick(hidingSpot) {
    if (this._isGameOver()) {
      return;
    }
    Relay.Store.commitUpdate(
      new CheckHidingSpotForTreasureMutation({
        game: this.props.game,
        hidingSpot,
      })
    );
  }
  _hasFoundTreasure() {
    return (
      this.props.game.hidingSpots.edges.some(edge => edge.node.hasTreasure)
    );
  }
  _isGameOver() {
    return !this.props.game.turnsRemaining || this._hasFoundTreasure();
  }
  renderGameBoard() {
    return this.props.game.hidingSpots.edges.map(edge => {
      return (
        <div
          key={edge.node.id}
          onClick={this._handleHidingSpotClick.bind(this, edge.node)}
          style={this._getHidingSpotStyle(edge.node)}
        />
      );
    });
  }
  render() {
    var headerText;
    if (this.props.relay.getPendingTransactions(this.props.game)) {
      headerText = '\u2026';
    } else if (this._hasFoundTreasure()) {
      headerText = 'You win!';
    } else if (this._isGameOver()) {
      headerText = 'Game over!';
    } else {
      headerText = 'Find the treasure!';
    }
    return (
      <div>
        <h1>{headerText}</h1>
        {this.renderGameBoard()}
        <p>Turns remaining: {this.props.game.turnsRemaining}</p>
      </div>
    );
  }
}

export default Relay.createContainer(App, {
  fragments: {
    game: () => Relay.QL`
      fragment on Game {
        turnsRemaining,
        hidingSpots(first: 9) {
          edges {
            node {
              hasBeenChecked,
              hasTreasure,
              id,
              ${CheckHidingSpotForTreasureMutation.getFragment('hidingSpot')},
            }
          }
        },
        ${CheckHidingSpotForTreasureMutation.getFragment('game')},
      }
    `,
  },
});
```

A working copy of the treasure hunt can be found under the [`./examples/`](https://github.com/facebook/relay/tree/master/examples) directory in the Relay repository.

Now that we've gone through this tutorial, let's dive into what it means to build a GraphQL client framework and how this compares to clients for more traditional REST systems.
