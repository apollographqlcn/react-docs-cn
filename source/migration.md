---
title: 从0.x迁移到1.0
sidebar_title: 迁移到1.0
description: 简易指南
---

以下是 Apollo 客户端从0.x到1.0版本之间的主要破坏性变更。

<h2 id="fetchMore">fetchMore</h2>

`fetchMoreResult` 的结构已经改变了。以前 `fetchMoreResult` 用于包含 `data` 和 `loading` 字段，现在的 `fetchMoreResult` 等同于之前的 `fetchMoreResult.data`。这意味着你的 `updateQueries` 函数必须针对 `fetchMore` 做如下修改：

```js
updateQuery: (prev, { fetchMoreResult }) => {
  return Object.assign({}, prev, {
    // feed：[... prev.feed，... fetchMoreResult.data.feed]，// 之前的写法
    feed: [...prev.feed, ...fetchMoreResult.feed], // 现在的写法
  });
},
```

<h2 id="fetchPolicy">fetchPolicy</h2>

`forceFetch` 和 `noFetch` 查询选项不再可用。相反，它们已被一个统一API `fetchPolicy` 替代。`fetchPolicy` 接收以下设置：

- `{ fetchPolicy: 'cache-first' }`: 这是 Apollo 客户端在没有指定获取策略时使用的默认策略。首先，它将尝试从缓存中完成查询。只有缓存查找失败，才将查询发送到服务器。
- `{ fetchPolicy: 'cache-only' }`: 使用此选项，Apollo 客户端将尝试仅从缓存中完成查询。如果缓存中没有任何数据可用，将抛出一个错误。这相当于之前的 `noFetch`。
- `{ fetchPolicy: 'network-only' }`: 使用此选项，Apollo 客户端将绕过缓存并直接将查询发送到服务器。这相当于之前的 `forceFetch`。
- `{ fetchPolicy: 'cache-and-network' }`: 使用此选项，Apollo 客户端将查询发送给服务器，但在服务器请求未完结时，从缓存返回数据，并在服务器响应到来后更新结果。

<h2 id="returnPartialData">returnPartialData</h2>

在 Apollo 1.0 中已经移除了 `returnPartialData` 查询选项，因为如果在此模式下运行查询，可能难以预测哪些数据可用。

要用 `returnPartialData` 替换执行单个查询的功能，建议运行两个单独的查询：

1. 一个大型查询，一旦视图加载，就请求此视图中需要显示的所有数据。
2. 相对于您已知的已被缓存的较大查询，仅获取其中一小部分的小型查询，然后可以在加载较大的查询时显示该查询的数据。

这里有一个例子：

```
const FullSomethingComponent => (props) => {
  if (props.data.loading) {
    return <PreviewSomethingComponent {...props} />
  }
}

const fullQuery = gql`{
  channel(name: "x") {
    name
    topic
    messages {
      text
    }
  }
}`;

const previewQuery = gql`{
  channel(name: "x") {
    name
    topic
  }
}`;

const PreviewSomethingComponent => (props) => {
  if (props.data.loading) {
    // 噢噢，我们没有相应数据，即使它应该在缓存中。因此只显示一个加载组件
    return (<Loading />);
  }
  
  // 只渲染通道名称和主题，显示加载 spinner 提示，或类似的东西。
  return (<div> ... stuff here </div>);

}

const SomethingComponentWithData = graphql(fullQuery)(FullSomethingComponent);

const PreviewSomethingComponentWithData = graphql(previewQuery)(PreviewSomethingComponent);

```

<h2 id="resultTransformer">resultTransformer</h2>

此全局选项允许在返回查询和突变结果之前对 Apollo 客户端进行转换。由于Apollo 客户端内部的逻辑很少被用到，以及复杂度很高，所以它已被移除。转换数据的推荐方法是将转换应用于 Apollo 客户端外部。
在 `react-apollo` 中，这可以在 `props` 选项中实现。

<h2 id="queryDeduplication">queryDeduplication</h2>

查询去重是 Apollo 客户端上的全局选项，可确保如果有多个相同的查询，Apollo 客户端将只向服务器发送一个。它通过在发送之前检查已经在运行中的查询的新查询来实现。

`queryDeduplication` 默认被设置为 `true`。可以通过将 `{queryDeduplication：false}` 传递给 Apollo 客户端构造函数来关闭它。

<h2 id="notifyOnNetworkStatusChange">notifyOnNetworkStatusChange</h2>

布尔值 `notifyOnNetworkStatusChange` 查询选项将在每次网络状态更改时触发一个新的可观察结果。
网络状态表明此查询是否有任何请求正在进行中，并提供该请求是何种类型（initial loading，refetch，setVariables，forceFetch）的信息。在之前的版本中，如果加载状态改变，Apollo 客户端将不会触发新的可观察结果。更多有关信息，请参阅 react-apollo 文档。

<h2 id="reduxRootKey">reduxRootKey</h2>

全局 `reduxRootKey` 选项已被弃用，现已被移除。现在应该使用 `reduxRootSelector` 来取代它。如果你不向 Apollo 提供你自己的 Redux Store，则不需要设置此选项。 `reduxRootSelector` 是可选的。
如果提供，它必须是一个返回 store 中 Apollo 部分的函数，如下所示：

```js

const client = new ApolloClient({
  reduxRootSelector: (store) => store['myCustomStoreKey'],
});
```
