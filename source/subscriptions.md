---
title: 订阅
---

除了使用查询获取数据和使用变量修改数据之外，GraphQL规范将很快获得第三种操作类型，称为 `订阅`。您可以[阅读GitHub上的RFC](https://github.com/facebook/graphql/blob/master/rfcs/Subscriptions.md)。虽然规范中的细微变化可能会在最终确定之前发生，但您可以在今天使用Apollo订阅。

GraphQL订阅是将数据从服务器推送到选择从服务器收听实时消息的客户端的一种方法。订阅与查询类似，因为它们指定要传递给客户端的一组字段，而不是立即返回单个答案，每次在服务器上发生特定事件时都会发送结果。

订阅的常见用例是向客户端通知特定事件，例如创建新对象，更新字段等。

<h2 id="overview">概述</h2>

GraphQL订阅必须在模式中定义，就像查询和突变：

```js
type Subscription {
  commentAdded(repoFullName: String!): Comment
}
```

在客户端上，订阅查询与任何其他操作类似：

```js
subscription onCommentAdded($repoFullName: String!){
  commentAdded(repoFullName: $repoFullName){
    id
    content
  }
}
```

发送给客户端的响应如下：

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

在上面的例子中，服务器被写入每次在GitHunt上添加一个特定存储库的注释时发送一个新的结果。请注意，上述代码仅在模式中定义GraphQL预订。阅读[在客户端上设置订阅](#subscriptions-client)和[设置服务器的GraphQL订阅](http://dev.apollodata.com/tools/graphql-subscriptions/index.html)，了解如何添加订阅您的应用程序

<h3 id="when-to-use">何时使用订阅</h3>

在大多数情况下，间歇性轮询或手动重新取样实际上是保持客户端更新的最佳方式。那么订阅什么时候最好的选择呢？订阅在以下情况下特别有用：

1. 初始状态很大，但增量变化集较小。可以使用查询获取起始状态，并随后通过订阅进行更新。
2. 在特定事件的情况下，您关心低延迟更新，例如在用户希望在几秒钟内接收新消息的聊天应用程序的情况下。

Apollo或GraphQL的未来版本可能包括对实时查询的支持，这将是一种低延迟的替代轮询方式，但在这一点上，GraphQL中的一般实时查询在一些相对实验的设置之外尚不可能。

<h2 id="subscriptions-client">客户端设置</h2>

GraphQL订阅最流行的传输方式是[`subscriptions-transport-ws`](https://github.com/apollographql/subscriptions-transport-ws)。此包由Apollo社区维护，但可以与任何客户端或服务器GraphQL实现一起使用。在本文中，我们将介绍如何在客户端上进行设置，但是您还需要一个服务器实现。服务，您可以[阅读关于如何使用JavaScript服务器的订阅](/tools/graphql-subscriptions/setup.html)，或者如果您使用GraphQL后端作为后端，即可享受开箱即用的订阅，比如[Graphcool](https://www.graph.cool/docs/tutorials/worldchat-subscriptions-example-ui0eizishe/)或[Scaphold](https://scaphold.io/blog/2016/11/09/build-realtime-apps-with-subs.html)。

我们来看看如何添加对这个传输的支持给Apollo Client。

首先，从npm安装`subscriptions-transport-ws`：

```shell
npm install --save subscriptions-transport-ws
```

然后，初始化一个GraphQL订阅传输客户机：

```js
import { SubscriptionClient } from 'subscriptions-transport-ws';

const wsClient = new SubscriptionClient(`ws://localhost:5000/`, {
  reconnect: true
});
```
然后，使用 `addGraphQLSubscriptions` 函数扩展现有的Apollo Client网络接口：

```js
import { ApolloClient, createNetworkInterface } from 'react-apollo';
import { SubscriptionClient, addGraphQLSubscriptions } from 'subscriptions-transport-ws';

// 创建一个普通的网络接口：
const networkInterface = createNetworkInterface({
  uri: 'http://localhost:3000/graphql'
});

// 使用WebSocket扩展网络接口
const networkInterfaceWithSubscriptions = addGraphQLSubscriptions(
  networkInterface,
  wsClient
);

// 最后，使用修改的网络接口创建您的ApolloClient实例
const client = new ApolloClient({
  networkInterface: networkInterfaceWithSubscriptions
});
```

现在，查询和突变将按照HTTP正常进行，但订阅将通过websocket传输完成。

<h2 id="subscribe-to-more">subscribeToMore</h2>

使用GraphQL订阅，您的客户端将从服务器推送提醒，您应该选择最适合您的应用程序的模式：

* 将其用作通知，并在其触发时运行所需的任何逻辑，例如提醒用户或重新获取数据
* 使用与通知一起发送的数据，并将其直接合并到商店（现有查询会自动通知）

用`subscribeToMore`，你可以轻松地做后者。

`subscribeToMore`是一个可用于 `react-apollo` 的每个查询结果的函数。它的工作原理就像[`fetchMore`](/reactions/cache-updates.html＃fetchMore)，不同之处在于每次订阅返回时都会调用更新函数，而不是只有一次。

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

现在，我们来添加订阅。

添加一个名为 `subscribeToNewComments` 的函数，它将使用`subscribeToMore` 订阅并使用`updateQuery`更新新数据的查询存储。

请注意，`updateQuery`回调必须返回与初始查询数据相同形状的对象，否则新数据将不会被合并。这里的新注释被推入`entry`的`comments`列表中：

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

并通过使用订阅变量调用`subscribeToNewComments`函数来启动实际订阅：

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

<h2 id="authentication">通过WebSocket验证</h2>

在许多情况下，必须先验证客户端，然后才能接收订阅结果。为此，`SubscriptionClient`构造函数接受一个`connectionParams`字段，该字段在设置任何订阅之前传递服务器可以用来验证连接的自定义对象。

```js
import {SubscriptionClient} from 'subscriptions-transport-ws';

const wsClient = new SubscriptionClient(`ws://localhost:5000/`, {
    reconnect: true,
    connectionParams: {
        authToken: user.authToken,
    },
});
```

> 您可以使用 `connectionParams` 作为您可能需要的其他任何内容，而不仅仅是身份验证，并使用[SubscriptionsServer](/tools/graphql-subscriptions/index.html)在服务器端检查其有效负载。
