---
sidebar_title: 综合
title: "API: 综合"
---

API 部分是包含 React Apollo 中所有功能的一份完整参考。如果你刚开始使用 React Apollo，那么你应该首先从[查询](queries.html)开始阅读“起步”章节，并在需要查找某个特定方法时回到此 API 参考。

<h2 id="core">核心</h2>

这些 API 不是 `React` 特定的，但是每个使用 Apollo 的 React 开发人员都需要了解它们。

<h3 id="gql">``gql`{ ... }` ``</h3>

```js
import { gql } from 'react-apollo';
```

`gql` 模板标签是你在 Apollo 客户端应用中定义 GraphQL 查询的方法。它将 GraphQL 查询解析成[GraphQL.js AST 格式] []，供 Apollo 客户端内置方法使用。每当 Apollo 客户端请求一个 GraphQL 查询时，你都应将其包裹在一个 `gql` 模板标签中。

你可以使用模板字符串插值将仅包含片段的 GraphQL 文档嵌入另一个 GraphQL 文档。这允许你在不同文件中定义的查询，能够使用别处定义的片段。如下代码演示了如何做到这一点。

为了方便起见，`gql` 标签在 `react-apollo` 中由 [`graphql-tag`][] 包重新导出。

[GraphQL.js AST格式]: https://github.com/graphql/graphql-js/blob/d92dd9883b76e54babf2b0ffccdab838f04fc46c/src/language/ast.js
[`graphql-tag`]: https://www.npmjs.com/package/graphql-tag

**例：**
请注意，在 `查询` 变量中，我们不仅通过模板字符串插值（`${fragments}`）包含了 `fragments` 变量，而且我们还在查询中包含了一个 `foo` 片段的扩展。

```js
const fragments = gql`
  fragment foo on Foo {
    a
    b
    c
    ...bar
  }

  fragment bar on Bar {
    d
    e
    f
  }
`;

const query = gql`
  query {
    ...foo
  }

  ${fragments}
`;
```

<h3 id="ApolloClient">`ApolloClient`</h3>

```js
import { ApolloClient } from 'react-apollo';
```

`ApolloClient` 实例是 Apollo API 的核心，它包含了你所需的与 GraphQL 数据交互的所有方法，无论你使用哪种集成，你都将使用该类。

要了解如何创建自己的 `ApolloClient` 实例，请参阅[初始化文档](initialization.html)。然后，将此实例传递给 [`<ApolloProvider />' 根组件](＃ApolloProvider)。

为了方便起见，`ApolloClient` 由 `react-apollo` 从 Apollo 客户端的核心包导出。

[要查看 `ApolloClient` 类的完整 API 文档，请访问核心文档站点。](/core/apollo-client-api.html#apollo-client)

**例：**

```js
const client = new ApolloClient({
  ...
});
```

<h3 id="createNetworkInterface">`createNetworkInterface(config)`</h3>

```js
import { createNetworkInterface } from 'react-apollo';
```

`createNetworkInterface()` 函数通过指定配置对象创建一个简单的 HTTP 网络接口，该对象包括 Apollo 用于请求 GraphQL 的 URI。

为了方便起见，`createNetworkInterface()` 由 Apollo 客户端的核心包 `react-apollo` 导出。

[了解更多关于 `createNetworkInterface` 和网络接口的信息，请访问核心文档站点。](/core/network.html)

**例：**

```js
const networkInterface = createNetworkInterface({
  uri: '/graphql',
});
```

<h2 id="client-management">客户端管理</h2>

React-Apollo 包含一个用于将客户端实例提供给 React 组件树的组件，以及用于检索该客户端实例的高阶组件。

<h3 id="ApolloProvider" title="ApolloProvider">`<ApolloProvider client={client} />`</h3>

```js
import { ApolloProvider } from 'react-apollo';
```

通过 `graphql()` 函数增强的任何组件都可以访问 GraphQL 客户端。 `<ApolloProvider />` 组件与[`react-redux` `<Provider />` 组件][]相同。它通过 [`graphql()`](#graphql) 函数或 [`withApollo`](#withApollo) 函数为所有 GraphQL 组件提供了一个 [`ApolloClient`][] 实例。你还可以使用 `<ApolloProvider />` 组件为你的 GraphQL 客户端提供额外的 Redux store。

如果你没有将此组件添加到你的 React 组件树的根组件上，那么使用 Apollo 的组件，其功能将无法正常工作。

要了解有关初始化 [`ApolloClient`][] 实例的更多信息，请务必阅读[设置和选项指南](initialization.html)。

`<ApolloProvider />`组件需要以下属性：

- `client`：所需的 [`ApolloClient`][] 实例。这个 [`ApolloClient`][] 实例将被所有使用 GraphQL 功能增强的组件所使用。
- `[store]`：这是 Redux store 的可选实例。如果你选择在这里传入 Redux store，那么 `<ApolloProvider />` 也会像[`react-redux` `<Provider />` 组件][]为你提供 Redux store。这意味着你只需要使用一个 provider 组件，而不是两个！

如果要直接访问你的组件中由 `<ApolloProvider />` 提供的 [`ApolloClient`][] 实例，那么请务必查看 [`withApollo()`](#withApollo) 增强器函数。

[`react-redux` `<Provider />` 组件]: https://github.com/reactjs/react-redux/blob/master/docs/api.md#provider-store
[`ApolloClient`]: /core/apollo-client-api.html#apollo-client

**例：**

```js
ReactDOM.render(
  <ApolloProvider client={client}>
    <MyRootComponent />
  </ApolloProvider>,
  document.getElementById('root'),
);
```

<h3 id="withApollo">`withApollo(component)`</h3>

```js
import { withApollo } from 'react-apollo';
```

作为一个简单的增强器，`withApollo()` 可以直接访问你的 [`ApolloClient`][] 实例。如果你想要使用 Apollo 处理自定义逻辑，它将会非常有用，如发起一次性查询。通过使用要增强的组件调用此函数，`withApollo()` 将创建一个新的组件，该组件的 `client` 属性接收 [`ApolloClient`][] 的一个实例。

如果你想知道什么时候使用 `withApollo()`，什么时候使用 [`graphql()`](#graphql)，答案是大多数时候你将使用 [`graphql()`](#graphql)。 [`graphql()`](#graphql) 提供了许多使用 GraphQL 数据所需的高级功能。如果你想要一个没有任何附加功能的 GraphQL 客户端，则应该使用 `withApollo()`。

如果你的组件树中有一个层级高于实际提供客户端实例的 [`<ApolloProvider/>`](#ApolloProvider) 组件，它将只能访问该客户端。

[`ApolloClient`]: /core/apollo-client-api.html#apollo-client

**例：**

```js
export default withApollo(MyComponent);

function MyComponent({ client }) {
  console.log(client);
}
```
