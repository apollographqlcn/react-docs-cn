---
title: 从0.x迁移到1.0
sidebar_title: 迁移到1.0
description: 简易指南。
---

以下是Apollo Client的0.x和1.0版本之间的主要突破性变化。

<h2 id="fetchMore">fetchMore</h2>

`fetchMoreResult`的结构已经改变了。以前`fetchMoreResult`用于包含`data`和`loading`字段，现在`fetchMoreResult`是`fetchMoreResult.data`以前是什么。这意味着你的`updateCheck`函数必须改变如下对于`fetchMore`：

```js
updateQuery: (prev, { fetchMoreResult }) => {
  return Object.assign({}, prev, {
    // feed：[... prev.feed，... fetchMoreResult.data.feed]，//这是以前是
    feed: [...prev.feed, ...fetchMoreResult.feed], //这是它现在要做的
  });
},
```

<h2 id="fetchPolicy">fetchPolicy</h2>

`forceFetch`和`noFetch`查询选项不再可用。相反，它们已被称为 `fetchPolicy` 的统一API替代。 `fetchPolicy`接受以下设置：

- `{ fetchPolicy: 'cache-first' }`: 这是Apollo Client在没有指定获取策略时使用的默认提取策略。首先，它将尝试从缓存中完成查询。只有缓存查找失败，才能将查询发送到服务器。
- `{ fetchPolicy: 'cache-only' }`: 使用此选项，Apollo Client将尝试仅从缓存中完成查询。如果缓存中没有全部数据可用，将抛出一个错误。这相当于前者的`noFetch`。
- `{ fetchPolicy: 'network-only' }`: 使用此选项，Apollo Client将绕过缓存并直接将查询发送到服务器。这相当于前面的`forceFetch`。
- `{ fetchPolicy: 'cache-and-network' }`: 使用此选项，Apollo Client将查询服务器，但在服务器请求未决时，从缓存返回数据，并在服务器响应到来后更新结果背部。

<h2 id="returnPartialData">returnPartialData</h2>

在 Apollo 1.0 中已经删除了 `returnPartialData` 查询选项，因为如果在此模式下运行查询，可能难以预测哪些数据可用。

要用 `returnPartialData` 替换运行一个查询的功能，建议运行两个单独的查询：

1. 一个大型查询，一旦加载，就要求在此视图中显示的所有数据。
2. 只读取您所知道的较大查询的一小部分的小型查询已被缓存。然后可以在加载较大的查询时显示该查询的数据。

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
    // 哎，我们没有这个数据，即使它应该在缓存中。只显示一个加载组件
    return (<Loading />);
  }
  
  // 只是渲染通道名称和主题，显示加载微调器的消息，或类似的东西。
  return (<div> ... stuff here </div>);

}

const SomethingComponentWithData = graphql(fullQuery)(FullSomethingComponent);

const PreviewSomethingComponentWithData = graphql(previewQuery)(PreviewSomethingComponent);

```

<h2 id="resultTransformer">resultTransformer</h2>

此全局选项允许在返回查询和变更结果之前对Apollo Client进行转换。由于Apollo Client内部的逻辑很少使用和复杂，所以它已被删除。推荐的转换数据的方法是将转换应用于Apollo Client外部。
在 `react-apollo` 中，这可以在 `props` 选项中完成。

<h2 id="queryDeduplication">queryDeduplication</h2>

查询重复数据删除是Apollo Client上的全局选项，可确保如果有多个相同的查询，Apollo Client将只向服务器发送一个。它通过在发送之前检查已经在运行中的查询的新查询来实现。

`queryDeduplication` 默认设置为 `true`。通过将`{queryDeduplication：false}`传递给Apollo Client构造函数可以关闭它。

<h2 id="notifyOnNetworkStatusChange">notifyOnNetworkStatusChange</h2>

布尔 `notifyOnNetworkStatusChange` 查询选项将在每次网络状态更改时触发新的可观察结果。
网络状态指示是否有任何请求正在飞行中用于此查询，并提供有关何种类型的请求（初始加载，重新读取，setVariables，forceFetch）的更多信息。在以前的版本中，如果加载状态改变，Apollo Client将不会在可观察的情况下触发新的结果。有关更多信息，请参阅react-apollo文档。

<h2 id="reduxRootKey">reduxRootKey</h2>

全局 `reduxRootKey` 选项已被弃用，现已被删除。 在它的位置应该使用 `reduxRootSelector`。 如果您不向阿波罗提供您自己的Redux Store，则不需要设置此选项。 `reduxRootSelector`是可选的。
如果提供，它必须是一个返回商店的阿波罗部分的功能，如下所示：

```js

const client = new ApolloClient({
  reduxRootSelector: (store) => store['myCustomStoreKey'],
});
```
