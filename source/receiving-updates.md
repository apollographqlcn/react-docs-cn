---
title: 同步服务端数据
sidebar_title: 同步服务端数据
---

Apollo 客户端缓存查询结果，然后在某些情况下使用此缓存来解析查询。您可以通过配置查询数据时使用的 [fetchPolicy](api-queries.html＃graphql-config-options-fetchPolicy) 来控制此行为。但是，如果我们的缓存中的信息过时会发生什么？如果服务端的信息发生变化，我们如何确保更新缓存？我们的UI将如何更新以反映这些新信息的变化？本节将尝试解答这些问题。

您可以设置一些策略，以确保 Apollo 客户端最终与服务端可用的信息保持一致。比如：手动查询，轮询查询和GraphQL订阅。

<h2 id="refetch">Refetch</h2>

`Refetch` 是强制部分缓存响应服务端可用信息的最简单方法。本质上，refetch 强制执行一个查询，立即再次请求服务端，绕过缓存。此查询的结果与所有其他查询结果一样，可更新缓存中可用的信息，从而更新页面上的所有查询结果。

例如，在 GitHunt 项目中，我们可能有如下组件实现：

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

假设我们在页面上有一个“刷新”按钮，当该按钮被点击时，在我们的组件上调用 `onRefreshClicked` 方法。我们有一个方法 `this.props.data.refetch`，它允许我们重新获取与 `feed` 组件相关联的查询。这意味着，不会从缓存中解析有关 `feed` 字段的信息，因此查询请求将直接到达服务端，并使用新的结果更新缓存。

如果服务端有某种更新（例如，添加到 Feed 中的新代码仓库），Apollo 客户端的 store 将获得更新，UI将根据需要重新呈现。

<h3 id="when-to-refetch">何时 refetch</h3>

为使 refetch 策略成功，您需要知道何时重新获取查询，所幸 refetch 的使用场景很少。

1. **手动刷新**：许多应用程序提供一种方式让用户通过刷新按钮或下拉手势手动请求更新其数据视图，`refetch` 非常适合这种情况。
2. **响应事件**：有时，您知道事件何时发生，将有可能会使数据无效。例如，您可能会有某种事件推送机制，告诉您数据可能已更改。因此，`refetch` 是可以避免不必要的轮询的最佳方式。
3. **突变后**：Apollo 提供了[一些方法](cache-updates.html)，以确保 store 在突变后是最新的，但最简单的方法是完全重新读取相关查询。在这种情况下，除了在查询本身使用 `refetch` 方法之外，还可以使用 [refetchQueries](api-mutations.html＃graphql-mutation-options-refetchQueries) 选项执行突变。

但是，在某些情况下，响应用户输入或更改 UI 的变更并无必要，例如，如果某些*其他*用户决定将代码仓库插入到 GitHunt Feed 中，并且我们需要展示这条数据。我们的客户不知道这已经发生了，直到刷新页面才会看到新的 Feed 项。该问题的解决方案之一便是轮询。

<h3 id="polling">轮询</h3>

如果您的查询结果可能会频繁变更，并且您想要查看相对较新的视图，则可以考虑轮询。轮询在特定的时间间隔内被触发，与 refetch 相似。

继续我们的 refetch 示例，我们可以通过在查询中添加一个选项来添加轮询间隔：

```javascript
const FeedWithData = graphql(FeedEntries, {
  options: { pollInterval: 20000 },
})(Feed);
```

查询的 `pollInterval` 选项设置该查询将被重新执行的时间间隔（以毫秒为单位）。在上述例子中，Apollo 会每隔二十秒处理一次这个查询，确保您的UI在任何时刻都只有20秒过时。

一般来说，您的轮询间隔不应小于10秒，否则容易对您的服务器造成大量负载。如果您需要立即在 UI 中反映数据变化，则应使用 [GraphQL 订阅](#subscriptions)。

## 订阅

订阅可让您的 UI 获得近实时更新。与轮询不同，订阅是基于推送的，这意味着服务端在更新可用时立即将其推送到客户端。订阅比轮询更难设置，但它们允许对更新进行更细粒度的控制，更新时间更快，并可能减少服务端的负载。

基于上面的 feedEntry 示例，我们可以通过添加对 `score` 字段的订阅来实时更新该字段：

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
          // 如果 Feed 已更改，我们需要在重新订阅之前取消订阅
          this.unsubscribe();
        } else {
          // 我们设置激活了一个带正确参数的订阅
          return;
        }
      }

      const entryIds = newProps.data.feed.map(item => item.id);

      this.unsubscribe = newProps.data.subscribeToMore({
        document: SUBSCRIPTION_QUERY,
        variables: { entryIds },

        // 奇迹发生的地方
        updateQuery: (previousResult, { subscriptionData }) => {
          const newScoreEntry = subscriptionData.data.newScore;

          const newResult = clonedeep(previousResult); // 不可变状态！

          // 更新受影响的条目的 score
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

在上面的示例中，我们通过订阅列出当前显示在组件中的 Feed 条目的所有 ID，来保留 Feed 组件中所有条目的 score 信息。每次更新 score 时，服务端将发送包含条目 ID 和新 score 的单个响应。

`subscribeToMore` 是一种通过订阅更新单个查询结果的方便的方法。每次新的订阅结果返回时，传递给 `subscribeToMore` 的 `updateQuery` 函数都会立即执行，它负责更新查询结果。`subscribeToMore` 返回取消订阅处理程序。

`subscribeToMore` 订阅在其依赖查询停止时自动停止，因此我们不需要手动取消订阅。如果 props 改变了，我们需要手动取消订阅，并使用不同的变量设置进行新的订阅。

要更深入地了解 GraphQL 中的订阅，您可以在我们的[博客文章](https://dev-blog.apollodata.com/graphql-subscriptions-in-apollo-client-9a2457f015fb)找到该主题。
