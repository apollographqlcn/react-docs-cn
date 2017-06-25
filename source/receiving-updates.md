---
title: 与服务器同步
sidebar_title: 与服务器同步
---

Apollo Client缓存查询结果，然后在可能的情况下使用此缓存来解析查询。您可以在要求查询数据时使用的[fetch policy](api-queries.html＃graphql-config-options-fetchPolicy)来控制此行为。但是，如果我们的缓存中的信息过时会发生什么？如果服务器上的信息发生变化，我们如何确保更新缓存？我们的UI将如何更新以反映这些新信息？本节将尝试回答这些问题。

您可以实施一些策略，以确保Apollo Client最终与服务器可用的信息保持一致。这些是：手动查询，轮询查询和GraphQL订阅。

<h2 id="refetch">Refetch</h2>

`Refetch` 是强制部分缓存反映服务器可用信息的最简单方法。本质上，refetch强制一个查询，立即再次击中服务器，绕过缓存。此查询的结果与所有其他查询结果一样，可更新缓存中可用的信息，从而更新页面上的所有查询结果。

例如，使用GitHunt模式，我们可能有以下组件实现：

```javascript
import React, { Component } from 'react';
import { graphql, gql } from 'react-apollo';

class Feed extends Component {
  // ...
  onRefreshClicked() {
    this.props.data.refetch();
  }
  // ...
}

const FeedEntries = gql`
  query FeedEntries($type: FeedType!, $offset: Int, $limit: Int) {
    feed($type: NEW, offset: $offset, limit: $limit) {
      createdAt
      commentCount
      score
      id
      respository {
        # etc.
      }
    }
  }
`;

const FeedWithData = graphql(FeedEntries)(Feed);
```

假设我们在页面上有一个“刷新”按钮，当该按钮被点击时，在我们的组件上调用`onRefreshClicked`方法。我们有一个方法`this.props.data.refetch`，它允许我们重新获取与`feed`组件相关联的查询。这意味着，不要从缓存中解析有关“feed”字段的信息，因此查询将被迫到达服务器，并使用新的结果更新缓存。

如果服务器上有某种更新（例如，添加到Feed中的新存储库），Apollo Client存储将获得更新，UI将根据需要重新呈现。

<h3 id="when-to-refetch">何时 refetch</h3>

为了获得成功的策略，您需要知道何时重新获取查询。很少有不同的使用案例，其中的重写是有意义的。

1. **手动刷新**：许多应用程序有一种方式让用户通过刷新按钮或下拉手势手动请求更新其数据视图。 `refetch`非常适合这种情况。
2. **响应事件**：有时，您知道事件何时发生，可能会使数据无效。例如，您可能会有某种事件推送，告诉您数据可能已更改。然后，`refetch`可以是避免不必要的轮询的最佳方式。
3. **突变后**：有[几种方法](cache-updates.html)，以确保商店在更改后是最新的，但最简单的方法是完全重新读取相关查询。在这种情况下，除了在查询本身使用`refetch`方法之外，还可以使用[refetchQueries](api-mutations.html＃graphql-mutation-options-refetchQueries)选项进行突变。

但是，在某些情况下，响应用户输入或更改UI的变更无助于帮助，例如，如果某些*其他*用户决定将存储库插入到GitHunt Feed中，并且我们要显示它。我们的客户不知道这已经发生了，直到刷新页面才会看到新的Feed项。该问题的一个解决方案是轮询。

<h3 id="polling">轮询</h3>

如果您的查询结果可能会频繁更改，并且您想要查看世界相对较新的视图，则考虑进行轮询查询是有意义的。轮询查询在特定的时间间隔内被触发，但是与引用相似。

继续我们的refetch示例，我们可以通过在查询中添加一个选项来添加轮询间隔：

```javascript
const FeedWithData = graphql(FeedEntries, {
  options: { pollInterval: 20000 },
})(Feed);
```

查询的`pollInterval`选项设置查询将被重新查询的时间间隔（以毫秒为单位）。在这种情况下，阿波罗会每隔二十秒处理一次这个查询，确保您的UI在任何时刻都只有20秒过时。

一般来说，您的轮询间隔不应小于10秒。这是在您的服务器上造成大量负载的简单方法。如果您需要立即在UI中反映的数据，则应使用[GraphQL 订阅](#subscriptions)。

## 订阅

订阅可让您在UI中获得近实时更新。与轮询不同，订阅是基于推送的，这意味着服务器在可用时立即将更新推送到客户端。订阅比轮询更难设置，但它们允许对更新进行更细粒度的控制，更快更新时间，并可能减少服务器上的负载。

基于上面的feedEntry示例，我们可以通过添加该字段的订阅来实时更新 `score` 字段：

```javascript

const SUBSCRIPTION_QUERY = gql`
  subscription scoreUpdates ($entryIds: [Int]){
    newScore(entryIds: $entryIds){
      id
      score
    }
  }
`;

class Feed extends Component {
  // ...
  componentWillReceiveProps(newProps) {
    if (!newProps.data.loading) {
      if (this.unsubscribe) {
        if (newProps.data.feed !== this.props.data.feed) {
          // 如果Feed已更改，我们需要在重新订阅之前取消订阅
          this.unsubscribe();
        } else {
          // 我们已经有了正确的参数的主动订阅
          return;
        }
      }

      const entryIds = newProps.data.feed.map(item => item.id);

      this.unsubscribe = newProps.data.subscribeToMore({
        document: SUBSCRIPTION_QUERY,
        variables: { entryIds },

        // 这是神奇的地方。
        updateQuery: (previousResult, { subscriptionData }) => {
          const newScoreEntry = subscriptionData.data.newScore;

          const newResult = clonedeep(previousResult); // 从不突变状态！

          // 更新受影响的条目的分数
          newResult.feed = newResult.feed.forEach(entry => {
            if(entry.id === newScoreEntry.id) {
              entry.score = newScoreEntry.score;
              return;
            }
          });

          return newResult;
        },
        onError: (err) => console.error(err),
      });
    }
  }
  // ...
}
```

在上面的示例中，我们通过订阅列出当前显示在组件中的Feed条目的所有ID，来保留Feed组件中所有条目的分数。每次更新分数时，服务器将发送包含条目ID和新分数的单个响应。

`subscribeToMore`是用订阅更新单个查询的结果的一种方便的方法。每次新的订阅结果到达时，`updateQuery`函数传递给`subscribeToMore`，它负责更新查询结果。
`subscribeToMore`返回取消订阅处理程序。

`subscribeToMore` 订阅在其依赖查询停止时自动停止，因此我们不需要手动取消订阅。如果道具改变了，我们需要手动取消订阅，我们需要使用不同的变量进行新的订阅。

要更深入地介绍GraphQL中的订阅，您可以在该主题中找到我们的[博客文章](https://dev-blog.apollodata.com/graphql-subscriptions-in-apollo-client-9a2457f015fb)。
