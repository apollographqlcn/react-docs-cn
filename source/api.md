---
sidebar_title: 综合
title: "API: 综合"
---

API部分是React Apollo中可用的每个功能的完整参考。如果你刚开始使用React Apollo，那么你应该首先从[查询](queries.html)开始阅读“使用”文章，并在需要查找特定方法时回到此API参考。

<h2 id="core">核心</h2>

这些API不是 `React` 特定的，但是每个使用Apollo的React开发人员都需要了解它们。

<h3 id="gql">``gql`{ ... }` ``</h3>

```js
import { gql } from 'react-apollo';
```

`gql` 模板标签是你在Apollo Client应用程序中定义GraphQL查询的方法。它将GraphQL查询解析成[GraphQL.js AST格式] []，然后可以由Apollo Client方法使用。每当Apollo Client要求一个GraphQL查询时，你将始终将其包装在一个 `gql` 模板标签中。

你可以使用模板字符串插值将仅包含片段的GraphQL文档嵌入另一个GraphQL文档。这允许你在完全不同的文件中的查询定义中使用在代码库的一部分中定义的片段。请参阅下面的示例来演示如何运行。

为了方便，`gql`标签从 `react-apollo` 中的[`graphql-tag`][]包中导出。

[GraphQL.js AST格式]: https://github.com/graphql/graphql-js/blob/d92dd9883b76e54babf2b0ffccdab838f04fc46c/src/language/ast.js
[`graphql-tag`]: https://www.npmjs.com/package/graphql-tag

**例：**
请注意，在 `查询` 变量中，我们不仅通过模板字符串插入（`$ {fragments}`）包含`fragments`变量，而且我们还在我们的查询中包含了一个`foo`片段的扩展。

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

`ApolloClient` 实例是Apollo API的核心。它包含你需要与GraphQL数据交互的所有方法，无论你使用哪种集成，你都将使用该类。

要了解如何创建自己的 `ApolloClient` 实例，请参阅[初始化文档文章](initialization.html)。然后，你将将此实例传递到根[`<ApolloProvider />'组件](＃ApolloProvider)。

为了方便起见，`ApolloClient` 由 `react-apollo` 从核心的Apollo Client包导出。

[要查看 `ApolloClient` 类的完整API文档，请访问核心文档站点。](../core/apollo-client-api.html#apollo-client)

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

`createNetworkInterface()`函数使用提供的配置对象创建一个简单的HTTP网络接口，该对象包括Apollo将用于从中获取GraphQL的URI。

为了方便起见，`createNetworkInterface()` 由核心Apollo Client包中的 `react-apollo` 导出。

[了解更多关于`createNetworkInterface`和网络接口的一般情况，请访问核心文档站点。](../core/network.html)

**例：**

```js
const networkInterface = createNetworkInterface({
  uri: '/graphql',
});
```

<h2 id="client-management">客户端管理</h2>

React-Apollo包括一个用于将客户机实例提供给React组件树的组件，以及用于检索该客户端实例的更高阶组件。

<h3 id="ApolloProvider" title="ApolloProvider">`<ApolloProvider client={client} />`</h3>

```js
import { ApolloProvider } from 'react-apollo';
```

使GraphView客户端可以通过`graphql()`函数增强的任何组件。 `<ApolloProvider/>` 组件与[`react-redux` `<Provider/>` component][]相同。它为你使用[`graphql()`](#graphql)函数或[`withApollo`](#withApollo)函数的所有GraphQL组件提供了一个[`ApolloClient`][]实例。你还可以使用 `<ApolloProvider/>` 组件为你的Redux store 提供除了提供GraphQL客户端。

如果你不将此组件添加到你的React树的根目录，那么使用Apollo功能增强的组件将无法正常工作。

要了解有关初始化[`ApolloClient`][]的实例的更多信息，请务必阅读[setup and options guide](initialization.html)。

`<ApolloProvider/>`组件需要以下属性：

- `client`：所需的[`ApolloClient`][]实例。这个[`ApolloClient`][]实例将被所有使用GraphQL能力增强的组件使用。
- `[store]`：这是Redux store的可选实例。如果你选择在这里传入Redux store，那么`<ApolloProvider />`也会像[`react-redux` `<Provider/>` component][]提供你的Redux store。这意味着你只需要使用一个提供者组件而不是两个！

如果要直接访问你的组件中由 `<ApolloProvider/>` 提供的[`ApolloClient`][]实例，那么请确保看看[`withApollo()`](#withApollo)增强器函数。

[`react-redux` `<Provider/>` 组件]: https://github.com/reactjs/react-redux/blob/master/docs/api.md#provider-store
[`ApolloClient`]: ../core/apollo-client-api.html#apollo-client

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

一个简单的增强器，可以直接访问你的[`ApolloClient`][]实例。如果你想要使用Apollo做自定义逻辑，这很有用。如拨打一次性查询。通过使用要增强的组件调用此函数，`withApollo()`将创建一个新的组件，该组件作为`client`参数传递到[`ApolloClient`][]的实例中。

如果你想知道什么时候使用`withApollo()`，什么时候使用[`graphql()`](#graphql)，答案是大多数时候你会想使用[`graphql()`](#graphql)。 [`graphql()`](#graphql) 提供了许多使用GraphQL数据所需的高级功能。如果你想要没有任何其他功能的GraphQL客户端，则应该只使用`withApollo()`。

如果你的组件树中有一个[`<ApolloProvider/>`](#ApolloProvider)组件高于实际提供客户端，这将只能提供访问你的客户端。

[`ApolloClient`]: ../core/apollo-client-api.html#apollo-client

**例：**

```js
export default withApollo(MyComponent);

function MyComponent({ client }) {
  console.log(client);
}
```
