---
sidebar_title: "graphql: 查询"
title: "API: 带查询的 graphql 容器"
---

> 本文详细介绍 `graphql()` 高阶组件的查询使用。要查看关于所有操作的选项，请参阅[一般graphql容器API文档](api-graphql.html)。

你传递给 `graphql()` 函数的操作决定了组件的行为方式。如果你将查询传递到 `graphql()` 函数中，那么你的组件将执行该查询并响应监听 store 中查询的更新。

使用 `graphql()` 函数执行查询的示例组件：

```js
export default graphql(gql`
  query TodoAppQuery {
    todos {
      id
      text
    }
  }
`)(TodoApp);

function TodoApp({ data: { todos } }) {
  return (
    <ul>
      {todos.map(({ id, text }) => (
        <li key={id}>{text}</li>
      ))}
    </ul>
  );
}
```

 有关 `graphql()` 函数查询的理论概述，请阅读[查询文档](queries.html)。要了解 `graphql()` 函数支持的所有有关查询的功能，请往下读。

<h2 id="graphql-query-data">`props.data`</h2>

使用 `graphql()` 创建的高阶组件会将一个 `data` 参数传给你的组件。像这样：

```js
render() {
  const { data } = this.props; // <- `data` 属性.
}
```

`data` 属性包含从查询中获取的数据，另外还有一些有用的属性和方法，可以用来控制 GraphQL 连接组件的生命周期。例如，我们有一个如下所示的查询：

```graphql
{
  viewer { name }
  todos { text }
}
```

你的 `data` prop 将会包含此数据：

```js
render() {
  const { data } = this.props;

  console.log(data.viewer); // <- 查询 `viewer` 返回的数据。
  console.log(data.todos); // <- 查询 `todos` 返回的数据。
}
```

`data` prop 还有一些有用的属性，可以直接从 `data` 对象访问。例如 `data.loading` 或 `data.error`。

具体来说，首先要确保在渲染之前始终检查你的组件中的 `data.loading` 和 `data.error` 属性，因为包含应用数据的 `data.todos` 等属性，在你的组件正在执行初始查询时可能尚未定义。检查 `data.loading` 和 `data.error` 可以帮助你避免任何未定义数据的问题。此类检查可能如下所示：

```js
render() {
  const { data: { loading, error, todos } } = this.props;
  if (loading) {
    return <p>Loading...</p>;
  } else if (error) {
    return <p>Error!</p>;
  } else {
    return (
      <ul>
        {todos.map(({ id, text }) => (
          <li key={id}>{text}</li>
        ))}
      </ul>
    );
  }
}
```

<h3 id="graphql-query-data-loading">`data.loading`</h3>

这是一个表示该组件当前是否正在执行查询请求的布尔值。该值为真时，意味着正在使用你的网络接口发送查询请求，并且还没有得到响应。一般使用此属性渲染 loading 组件。

但是，如果 `data.loading` 值为 true，并不意味着你没有数据。举个例子，如果你已经有 `data.todos` 数据，但是你想从你的 API 中获取最新的 todos，此时，`data.loading` 可能为真，但是你仍然能访问你之前请求的 todos。

你请求的查询可能存在多种不同的网络状态。如果要更详细地了解组件的网络状态，请参阅 [`data.networkStatus`](#graphql-query-data-networkStatus)。

**例：**

```js
function MyComponent({ data: { loading } }) {
  if (loading) {
    return <div>Loading...</div>;
  } else {
    // ...
  }
}

export default graphql(gql`query { ... }`)(MyComponent);
```

<h3 id="graphql-query-data-error">`data.error`</h3>

如果请求发生错误，那么该属性将被赋值为 [`ApolloError`][] 的一个实例。如果你不处理此错误，你将在控制台中收到一条警告信息：`"Unhandled (in react-apollo) Error: ..."`。

[`ApolloError`]: /core/apollo-client-api.html#ApolloError

**例：**

```js
function MyComponent({ data: { error } }) {
  if (error) {
    return <div>Error!</div>;
  } else {
    // ...
  }
}

export default graphql(gql`query { ... }`)(MyComponent);
```

<h3 id="graphql-query-data-networkStatus">`data.networkStatus`</h3>

如果要根据网络状态显示不同的加载指示符（或根本没有指示符），则 `data.networkStatus` 非常有用，因为它为组件提供了比 [`data.loading`](#graphql-query-data-loading) 更详细的有关网络状态的信息。`data.networkStatus` 的值类型是一个1至8之间不同数字值的枚举，这些数值各自表示不同的网络状态。

1. `loading`：查询运行之后，且请求仍处于待处理状态。即使在高速缓存中命中结果，查询请求仍然具有此网络状态，但是查询已被调度。
2. `setVariables`：如果查询的变量发生变化，且网络请求被触发，则网络状态为 `setVariables`，直到该查询的结果返回。当 [`options.variables`](＃graphql-query-options-variables) 对查询进行更改时，React 用户会感受到这一点。
3. `fetchMore`：表示在这个查询中调用了 `fetchMore`，并且所创建的网络请求当前正在运行。
4. `refetch`：这意味着 `refetch` 在该查询中被调用，并且 refetch 请求当前正在运行。
5. 未用到。
6. `poll`：表示当前轮询查询正在运行。因此，假设如果你每10秒钟轮询一次查询，那么当轮询请求发送但未解决时，网络状态将每10秒切换到 `poll`。
7. `ready`：当前查询没有请求正在运行，且没有发生错误，一切安好。
8. `error`：当前查询没有请求正在运行，但检测到一个或多个错误。

如果网络状态小于7，则相当于 [`data.loading`](#graphql-query-data-loading) 为真。实际上，你可以用 `data.networkStatus < 7` 替换所有的 `data.loading` 检查，尽管没有什么区别，但建议你使用 `data.loading`。

**例：**

```js
function MyComponent({ data: { networkStatus } }) {
  if (networkStatus === 6) {
    return <div>Polling!</div>;
  } else if (networkStatus < 7) {
    return <div>Loading...</div>;
  } else {
    // ...
  }
}

export default graphql(gql`query { ... }`)(MyComponent);
```

<h3 id="graphql-query-data-variables">`data.variables`</h3>

Apollo 用来从 GraphQL 端点获取数据的变量。如果要根据向服务端发送请求的变量呈现某些信息，此属性将非常有用。

**例：**

```js
function MyComponent({ data: { variables } }) {
  return (
    <div>
      Query executed with the following variables:
      <code>{JSON.stringify(variables)}</code>
    </div>
  );
}

export default graphql(gql`query { ... }`)(MyComponent);
```

<h3 id="graphql-query-data-refetch">`data.refetch(variables)`</h3>

强制你的组件重新执行你在 `graphql()` 函数中定义的查询。当你要重新加载组件中的数据，或者在错误发生后重试请求时，此方法将会很有用。

`data.refetch` 返回一个 promise，一旦查询执行结束，就会将你的 API 中获取的新数据解析出来。如果查询失败，promise 将会 reject。

`data.refetch` 函数接收一个 `variables` 对象参数。`variables` 参数将替换查询选项或 `graphql（）` 高阶组件（根据你是否指定了一个 `query`）选项来重新请求你在 `graphql（）` 函数中定义的查询。

**例：**

```js
function MyComponent({ data: { refetch } }) {
  return (
    <button onClick={() => refetch()}>
      Reload
    </button>
  );
}

export default graphql(gql`query { ... }`)(MyComponent);
```

<h3 id="graphql-query-data-fetchMore">`data.fetchMore(options)`</h3>

`data.fetchMore` 功能可以让你对查询组件进行分页。要了解更多有关使用 `data.fetchMore` 分页的信息，请务必阅读[分页](pagination.html)说明，其中包含如何使用 React Apollo 进行分页的有用插图。

`data.fetchMore` 返回一个 Promise，一旦请求更多数据的查询 resolve，该 Promise 立即 resolve。

`data.fetchMore` 函数使用一个 `options` 对象作为参数。`options` 参数可能具有以下属性：

- `[query]`：这是使用 `gql` GraphQL 标签创建的可选 GraphQL 文档。如果你指定了一个 `query`，那么当你调用 `data.fetchMore` 的时候，该查询将被执行。如果没有指定 `query`，那么将使用 `graphql()` 高阶组件的查询。
- `[variables]`：作为一个可选变量，将与 `query` 选项或 `graphql()` 高阶组件中的查询一起使用（取决于你是否指定了一个 `query`）。
- `updateQuery(previousResult, { fetchMoreResult, queryVariables })`：这是由你定义的必要函数，将会实际更新你的分页列表。第一个参数`previousResult` 将会是你在 `graphql()` 函数中定义的查询返回的之前的数据。第二个参数是一个具有两个属性 `fetchMoreResult` 和 `queryVariables` 的对象。`fetchMoreResult` 是新请求返回的数据，它由 `data.fetchMore` 的 `query` 和 `variables` 选项得来。 `queryVariables` 是请求更多数据时使用的变量。通过指定这些参数，你将会得到一个服务端返回的新的数据对象，该对象与你在 `graphql()` 函数中定义的 GraphQL 查询具有相同形式。请参考下面的示例，并确保阅读具有完整示例的[分页](pagination.html) 说明。

**例：**

```js
data.fetchMore({
  updateQuery: (previousResult, { fetchMoreResult, queryVariables }) => {
    return {
      ...previousResult,
      // 将新的 Feed 数据添加到旧的 Feed 数据的末尾。
      feed: [...previousResult.feed, ...fetchMoreResult.feed],
    };
  },
});
```

<h3 id="graphql-query-data-subscribeToMore">`data.subscribeToMore(options)`</h3>

该函数将设置一个订阅，每当服务器发送订阅发布时将触发更新。这需要在服务器上设置订阅才能正常工作。查看[订阅指南](receiving-updates.html#Subscriptions) ，[subscriptions-transport-ws](https://github.com/apollographql/subscriptions-transport-ws) 和 [graphql-subscriptions](https://github.com/apollographql/graphql-subscriptions) 获取更多相关信息。

该函数返回一个 `unsubscribe` 函数处理器，用以在之后取消订阅。

通常的做法是将 `subscribeToMore` 封装到 `componentWillReceiveProps` 中供组件调用，并在原始查询完成后执行订阅。为确保订阅不会被多次创建，你可以将其附加到组件实例。更多详细信息，请参考示例。

- `[document]`：Document 是一个必需的属性，它接受用 `graphql-tag` 的 `gql` 模板字符串标签创建的 GraphQL 订阅文档。它应该包含一个指定返回数据的 GraphQL 订阅操作。
- `[variables]`：作为可选变量将与 `document` 选项一起使用。
- `[updateQuery]`：每次服务器发送更新时运行的可选函数。它将修改高阶组件查询的结果。第一个参数 `previousResult` 将是你在 `graphql()` 函数中定义的查询之前返回的数据。第二个参数是一个具有两个属性的对象。`subscriptionData`是订阅的结果。 `variables` 是与订阅查询一起使用的变量对象。使用这些参数，你应该返回一个与你在 `graphql()` 函数中定义的 GraphQL 查询相同形状的新的数据对象。这与 [`fetchMore`](#graphql-query-data-fetchMore) 回调类似。或者，你可以使用 [reducer](cache-updates.html#resultReducers) 作为你的 `graphql()` 函数的[options](queries.html#graphql-options) 的一部分来更新查询。
- `[onError]`：一个可选的错误回调。

如果要使用订阅结果更新查询的 store，你必须在 `graphql()` 函数中指定 `subscribeToMore` 的 `updateQuery` 选项或 `reducer` 选项。

**例：**

```js
class SubscriptionComponent extends Component {
  componentWillReceiveProps(nextProps) {
    if(!nextProps.data.loading) {
      // 检查现有订阅
      if (this.unsubscribe) {
        // 检查属性是否已更改，如有必要，请停止订阅
        if (this.props.subscriptionParam !== nextProps.subscriptionParam) {
          this.unsubscribe();     
        } else {
          return;
        }
      }
    
      // 订阅
      this.unsubscribe = nextProps.data.subscribeToMore({
        document: gql`subscription {...}`,
        updateQuery: (previousResult, { subscriptionData, variables }) => {
          // 使用 subscriptionData 执行 previousResult 的更新
          return updatedResult;
        }
      });
    }
  }
  render() {
    ...
  }
}
```


<h3 id="graphql-query-data-startPolling">`data.startPolling(interval)`</h3>

该函数将设置一个时间间隔，每隔一段固定时间便发送请求。该函数只有一个整数参数，允许你配置以毫秒为单位执行查询的频率。换句话说，`interval` 参数表示轮询之间的毫秒数。

轮询是将UI中的数据保持更新的好方法。通过每5000毫秒（例如5秒钟）重新获取数据，你可以有效地模拟实时数据，而无需建立实时后端。

当查询已经在执行轮询时，如果你调用 `data.startPolling`，那么当前的轮询操作将被取消，并且以指定的时间间隔开始一个新的轮询操作。

你也可以使用 [`options.pollInterval`](#graphql-config-options-pollInterval)，在你的组件挂载后立即开始轮询。如果你不需要任意地启动和停止轮询，建议你使用 [`options.pollInterval`](#graphql-config-options-pollInterval)。

如果你将 `interval` 设置为0，那么这意味着没有轮询，而不是每个 JavaScript 事件循环滴答均执行一次请求。

**例：**

```js
class MyComponent extends Component {
  componentDidMount() {
    // 在这种具体情况下，你可能需要使用 `options.pollInterval`。
    this.props.data.startPolling(1000);
  }

  render() {
    // ...
  }
}

export default graphql(gql`query { ... }`)(MyComponent);
```

<h3 id="graphql-query-data-stopPolling">`data.stopPolling()`</h3>

通过调用此函数，你可以终止任意当前正在执行的轮询操作。在你调用 `data.startPolling` 之前，你的查询将不会再次发起轮询。

**例：**

```js
class MyComponent extends Component {
  render() {
    return (
      <div>
        <button onClick={() => {
          this.props.data.startPolling(1000);
        }}>
          Start Polling
        </button>
        <button onClick={() => {
          this.props.data.stopPolling();
        }}>
          Stop Polling
        </button>
      </div>
    )
  }
}

export default graphql(gql`query { ... }`)(MyComponent);
```

<h3 id="graphql-query-data-updateQuery">`data.updateQuery(updaterFn)`</h3>

该函数允许你在任何突变，订阅或请求的上下文之外更新查询的数据。该函数只接收另一个函数作为它单独的参数。函数参数具有以下特征：

```
(previousResult, { variables }) => nextResult
```

第一个参数是 store 中当前存在的查询的数据，并且你期望返回具有相同形状的新数据对象。该新数据对象将被写入 store，并且跟踪该数据的任何组件都将会响应更新。

第二个参数是一个具有单个属性 `variables` 的对象。`variables` 属性允许你从 store 中读取 `previousResult` 时查看其使用的变量。

此方法*不会*更新服务器上的任何东西。它只会更新客户端缓存中的数据，如果你重新加载 JavaScript 环境，则更新将消失。

**例：**

```js
data.updateQuery((previousResult) => ({
  ...previousResult,
  count: previousResult.count + 1,
}));
```

<h2 id="graphql-query-options">`config.options`</h2>

返回用于配置如何获取和更新查询的选项对象的对象或函数。

如果 `config.options` 是一个函数，那么它将以组件的 props 为第一个参数。

可用于此对象的选项取决于你为 `graphql()` 的第一个参数传入的操作类型。以下参考文档将记录你的操作是查询时哪些选项可用。要查看其他用于不同操作的选项，请参阅 [`config.options`](api-graphql.html#graphql-config-options) 的通用文档。

**例：**

```js
export default graphql(gql`{ ... }`, {
  options: {
    // 在这里填写选项。
  },
})(MyComponent);
```

```js
export default graphql(gql`{ ... }`, {
  options: (props) => ({
    // 从这里的`props`计算选项。
  }),
})(MyComponent);
```

<h3 id="graphql-config-options-variables">`options.variables`</h3>

The variables that will be used when executing the query operation. These variables should correspond with the variables that your query definition accepts. If you define config.options as a function then you may compute your variables from your props.

执行查询操作时将使用的变量。这些变量应与查询定义接受的变量相对应。如果你定义 `config.options` 为一个函数，那么你可以从你的 props 计算你的变量。

**例：**

```js
export default graphql(gql`
  query ($width: Int!, $height: Int!) {
    ...
  }
`, {
  options: (props) => ({
    variables: {
      width: props.size,
      height: props.size,
    },
  }),
})(MyComponent);
```

<h3 id="graphql-config-options-fetchPolicy">`options.fetchPolicy`</h3>

`fetchPolicy` 是一个选项，可让你指定组件如何与 Apollo 数据缓存进行交互。默认情况下，你的组件将首先尝试从缓存中读取数据，如果查询的完整数据位于缓存中，则 Apollo 将从缓存中直接返回该数据。如果你的查询的完整数据*不*在缓存中，则 Apollo 将使用你的网络接口执行你的请求。通过更改此选项，你可以更改默认行为。

`fetchPolicy` 有效值有：

- `cache-first`：作为默认值，在该策略下，我们始终尝试从缓存中读取数据。如果完成查询所需的所有数据都在缓存中，那么该数据将被返回。如果缓存结果不可用，Apollo 将只会从网络中请求。该请求策略旨在最大限度地减少渲染组件时发送的网络请求数。
- `cache-and-network`：该请求策略将让 Apollo 首先尝试从缓存中读取数据。如果完成查询所需的所有数据都在缓存中，那么该数据将被返回。但是，无论你的缓存中是否存在完整数据，`fetchPolicy` 将*始终*通过网络接口执行查询，不同于 `cache-first`，只有当查询数据不在缓存中时，才执行查询。该请求策略优化使得用户获得快速响应，同时尝试将高速缓存的数据与服务器数据保持一致，但需要额外的网络请求。
- `network-only`：该请求策略将*永远不会*从缓存中返回初始数据。相反，它将始终使用你的网络接口向服务器发出请求。该请求策略优化是为了与服务器的数据保持一致，但是牺牲了响应在可用时立即返回给用户的及时性。
- `cache-only`：该请求策略将*永远不会*使用你的网络接口执行查询。相反，它将始终尝试从缓存中读取数据。如果查询的数据不存在于缓存中，则会抛出错误。该请求策略允许你只与本地客户端缓存中的数据进行交互，而不会发生任何网络请求，从而保持组件快速响应，但意味着你的本地数据可能与服务器上的数据不一致。如果你只对 Apollo Client 缓存中的数据进行交互感兴趣，那么请务必查阅你的 [`ApolloClient`][] 实例中可用的 [`readQuery()`][] 和 [`readFragment()`][] 方法。

[`readQuery()`]: /core/apollo-client-api.html#ApolloClient.readQuery
[`readFragment()`]: /core/apollo-client-api.html#ApolloClient.readFragment
[`ApolloClient`]: /core/apollo-client-api.html#apollo-client

**例：**

```js
export default graphql(gql`query { ... }`, {
  options: { fetchPolicy: 'cache-and-network' },
})(MyComponent);
```

<h3 id="graphql-config-options-pollInterval">`options.pollInterval`</h3>

表示你要开始轮询的间隔（以毫秒为单位）。无论何时经过这个毫秒数，你的查询将使用网络接口执行，下一个执行将使用配置的毫秒数进行调度。

当组件挂载时，此选项将立即开始轮询你的查询。如果要动态启动和停止轮询，则可以使用 [`data.startPolling`](#graphql-query-data-startPolling) 和 [`data.stopPolling`](#graphql-query-data-stopPolling)。

如果将 `options.pollInterval` 设置为0，那么这意味着不会执行轮询，每个 JavaScript 事件循环滴答都不会执行请求。

**例：**

```js
export default graphql(gql`query { ... }`, {
  options: { pollInterval: 5000 },
})(MyComponent);
```

<h3 id="graphql-config-options-notifyOnNetworkStatusChange">`options.notifyOnNetworkStatusChange`</h3>

更新网络状态或网络错误是否会触发组件的重新渲染。

默认值为 `false`。

**例：**

```js
export default graphql(gql`query { ... }`, {
  options: { notifyOnNetworkStatusChange: true },
})(MyComponent);
```
