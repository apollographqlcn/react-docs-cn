---
title: 订阅
---

除了使用查询获取数据和使用突变修改数据之外，GraphQL 规范将很快制定第三种操作类型，称为 `subscription（订阅）`。您可以[阅读 GitHub 上的 RFC](https://github.com/facebook/graphql/blob/master/rfcs/Subscriptions.md)。虽然规范可能会在最终定稿之前发生细微变化，但您可以现在就使用 Apollo 实现订阅功能。

GraphQL 订阅是将数据从服务端推送到选择从服务端监听实时消息的客户端的一种方法。订阅与查询类似，因为它们都指定一组传递给客户端的字段，而不是立即返回单个答案，每次在服务器上发生特定事件时都会发送结果。

订阅常见的使用场景是向客户端通知特定事件，例如创建新对象，更新字段等。

<h2 id="overview">概述</h2>

GraphQL 订阅必须在 Schema 中定义，和查询和突变一样：

```js
type Subscription {
  commentAdded(repoFullName: String!): Comment
}
```

在客户端上，订阅的查询语句与其他操作类似：

```js
subscription onCommentAdded($repoFullName: String!){
  commentAdded(repoFullName: $repoFullName){
    id
    content
  }
}
```

客户端收到的响应如下：

```json
{
  "data": {
    "commentAdded": {
      "id": "123",
      "content": "Hello!"
    }
  }
}
```

在上面的例子中，每向 GitHunt 上指定代码仓库添加一条评论，服务端将发送一个新的响应结果。请注意，上述代码仅在 Schema 中定义 GraphQL 订阅。阅读[在客户端上设置订阅](#subscriptions-client)和[设置服务端的 GraphQL 订阅](http://dev.apollodata.com/tools/graphql-subscriptions/index.html)，了解如何为您的应用添加订阅。

<h3 id="when-to-use">何时使用订阅</h3>

在大多数情况下，保持客户端更新的最佳方式实际上是使用轮询或手动调用。那什么时候订阅会是更好的选择呢？通常订阅在以下情况下特别有用：

1. 初始状态数据很大，但增量变化集较小。可以使用查询获取初始状态，然后通过订阅进行更新。
2. 在某些特定的情况下，您比较在意低延迟更新，比如用户期望能在几秒钟内接收新消息的聊天应用。

Apollo 或 GraphQL 的未来版本可能包括对实时查询的支持，这将是一种低延迟的替代轮询的方案，但目前而言，GraphQL 中一般的实时查询只支持一些偏实验的设置。

<h2 id="subscriptions-client">客户端设置</h2>

GraphQL 订阅最流行的传输方式是 [`subscriptions-transport-ws`](https://github.com/apollographql/subscriptions-transport-ws)。此包由 Apollo 社区开发维护，但可以与任何客户端或服务端 GraphQL 实现一起使用。在本文中，我们将介绍如何在客户端进行设置，但前提是您需要有一个服务端实现。您可以[阅读关于如何使用JavaScript 服务端订阅](/tools/graphql-subscriptions/setup.html)，或者如果您使用了 GraphQL 后端服务，比如 [Graphcool](https://www.graph.cool/docs/tutorials/worldchat-subscriptions-example-ui0eizishe/) 或 [Scaphold](https://scaphold.io/blog/2016/11/09/build-realtime-apps-with-subs.html)，即可享受开箱即用的订阅。

我们来看看如何向 Apollo 客户端添加对这个传输方式的支持。

首先，从 npm 安装`subscriptions-transport-ws`：

```shell
npm install --save subscriptions-transport-ws
```

然后，初始化一个 GraphQL 订阅传输在客户端的实例：

```js
import { SubscriptionClient } from 'subscriptions-transport-ws';

const wsClient = new SubscriptionClient(`ws://localhost:5000/`, {
  reconnect: true
});
```
然后，使用 `addGraphQLSubscriptions` 函数扩展现有的 Apollo 客户端网络接口：

```js
import { ApolloClient, createNetworkInterface } from 'react-apollo';
import { SubscriptionClient, addGraphQLSubscriptions } from 'subscriptions-transport-ws';

// 创建一个普通的网络接口：
const networkInterface = createNetworkInterface({
  uri: 'http://localhost:3000'
});

// 使用 WebSocket 扩展网络接口
const networkInterfaceWithSubscriptions = addGraphQLSubscriptions(
  networkInterface,
  wsClient
);

// 最后，使用修改的网络接口创建您的 ApolloClient 实例
const client = new ApolloClient({
  networkInterface: networkInterfaceWithSubscriptions
});
```

现在，查询和突变将按照 HTTP 正常进行，但订阅将通过 websocket 传输完成。

<h2 id="subscribe-to-more">subscribeToMore</h2>

使用 GraphQL 订阅，您的客户端将从服务器获取推送提醒，您应该选择最适合您的应用程序的模式：

* 将其用作通知，并在其触发时运行所需的任何逻辑，例如提醒用户或重新获取数据
* 使用与通知一起发送的数据，并将其直接合并到 store（既有查询会自动通知）

用 `subscribeToMore`，您可以轻松地实现后者。

`subscribeToMore` 是一个可用于 `react-apollo` 的每个查询结果的函数。它的工作原理就像 [`fetchMore`](/reactions/cache-updates.html＃fetchMore)，不同之处在于每次订阅结果返回时都会调用更新函数，而不是只调用一次。

这是常规查询：

```js
import { CommentsPage } from './comments-page.js';
import { graphql } from 'react-apollo';
import gql from 'graphql-tag';

const COMMENT_QUERY = gql`
    query Comment($repoName: String!) {
      entry(repoFullName: $repoName) {
        comments {
          id
          content
        }
      }
    }
`;

const withData = graphql(COMMENT_QUERY, {
    name: 'comments',
    options: ({ params }) => ({
        variables: {
            repoName: `${params.org}/${params.repoName}`
        },
    })
});

export const CommentsPageWithData = withData(CommentsPage);
```

现在，我们来为其添加订阅。

添加一个名为 `subscribeToNewComments` 的函数，它将使用 `subscribeToMore` 订阅，并使用 `updateQuery` 获取的新数据更新查询 store。

请注意，`updateQuery` 回调必须返回与初始查询数据相同结构的对象，否则新数据将不会被合并。这里的新注释被推入 `entry` 的 `comments` 列表中：

```js
const COMMENTS_SUBSCRIPTION = gql`
    subscription onCommentAdded($repoFullName: String!){
      commentAdded(repoFullName: $repoFullName){
        id
        content
      }
    }
`;

const withData = graphql(COMMENT_QUERY, {
    name: 'comments',
    options: ({ params }) => ({
        variables: {
            repoName: `${params.org}/${params.repoName}`
        },
    }),
    props: props => {
        return {
            subscribeToNewComments: params => {
                return props.comments.subscribeToMore({
                    document: COMMENTS_SUBSCRIPTION,
                    variables: {
                        repoName: params.repoFullName,
                    },
                    updateQuery: (prev, {subscriptionData}) => {
                        if (!subscriptionData.data) {
                            return prev;
                        }

                        const newFeedItem = subscriptionData.data.commentAdded;

                        return Object.assign({}, prev, {
                            entry: {
                                comments: [newFeedItem, ...prev.entry.comments]
                            }
                        });
                    }
                });
            }
        };
    },
});
```

通过使用订阅变量调用 `subscribeToNewComments` 函数来启动实际订阅：

```js
export class CommentsPage extends Component {
    static propTypes = {
        repoFullName: PropTypes.string.isRequired,
        subscribeToNewComments: PropTypes.func.isRequired,
    }

    componentWillMount() {
        this.props.subscribeToNewComments({
            repoFullName: this.props.repoFullName,
        });
    }
}
```

<h2 id="authentication">通过 WebSocket 认证</h2>

在许多情况下，必须先对客户端进行认证，然后才能接收订阅结果。为此，`SubscriptionClient` 构造函数接受一个 `connectionParams` 字段，该字段传递一个自定义对象，在设置任何订阅之前可以被服务端用来验证连接。

```js
import {SubscriptionClient} from 'subscriptions-transport-ws';

const wsClient = new SubscriptionClient(`ws://localhost:5000/`, {
    reconnect: true,
    connectionParams: {
        authToken: user.authToken,
    },
});
```

> 除了身份认证，您还可以使用 `connectionParams` 设置您可能需要的其他配置选项，并使用 [SubscriptionsServer](/tools/graphql-subscriptions/index.html) 在服务端检查其有效性。
