---
sidebar_title: "graphql: 查询"
title: "API: 具有查询的graphql容器"
---

> 本文特别关于使用 `graphql()` 更高阶组件的查询。要查看适用于所有操作的选项，请参阅[一般graphql容器API文档](/react/api-graphql.html)。

您传递到 `graphql()` 函数的操作决定了组件的行为方式。如果您将查询传递到`graphql()`函数中，那么您的组件将获取该查询并反应性地侦听商店中查询的更新。

使用`graphql()`函数的查询的示例组件：

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

要更自然地使用 `graphql()` 函数查询查询，请务必阅读[查询文档文章](queries.html)。有关查询的 `graphql()` 函数支持的所有功能的技术概述，请继续。

<h2 id="graphql-query-data">`props.data`</h2>

在使用 `graphql()` 时创建的高阶组件会将一个`data`的参数提供给你的组件。像这样：

```js
render() {
  const { data } = this.props; // <- `data` 属性.
}
```

`data` 属性包含从查询中提取的数据，另外还有一些其他有用的信息和功能来控制GraphQL连接的组件的生命周期。所以例如，如果我们有一个如下所示的查询：

```graphql
{
  viewer { name }
  todos { text }
}
```

你的 `data` 属性会包含以下数据：

```js
render() {
  const { data } = this.props;

  console.log(data.viewer); // <- 您的查询为 `viewer` 返回的数据。
  console.log(data.todos); // <- 您的查询为 `todos` 返回的数据。
}
```

`data` 属性有一些其他有用的属性，可以直接从`data`访问。例如`data.loading`或`data.error`。这些属性如下所述。

确保在渲染之前始终检查您的组件中的 `data.loading` 和 `data.error`。包含应用程序数据的 `data.todos` 等属性可能在您的组件正在执行初始抓取时未定义。检查 `data.loading` 和 `data.error` 可以帮助您避免任何未定义数据的问题。此类检查可能如下所示：

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

一个布尔值，表示该组件当前是否正在运行查询请求。这意味着使用您的网络接口发送查询请求，我们还没有得到回复。使用此属性渲染加载组件。

但是，只是因为`data.loading`值为true，这并不意味着你不会有数据。例如，如果你已经有`data.todos`，但是你想从你的API中获取最新的todos`data.loading`可能是真的，但是你仍然会有你之前的请求的todos。

您的查询可能存在多种不同的网络状态。如果要查看组件的网络状态更详细，请参阅[`data.networkStatus`](#graphql-query-data-networkStatus)。

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

如果发生错误，那么该属性将是[`ApolloError`][]的一个实例。如果您不处理此错误，您将在控制台中收到一条警告信息：`"Unhandled (in react-apollo) Error: ..."`。

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

如果要根据网络状态显示不同的加载指示符（或根本没有指示符），则 `data.networkStatus` 非常有用，因为它提供了与[`data.loading`](#graphql-query-data-loading)。 `data.networkStatus`是一个在1到8之间具有不同数字值的枚举。这些数值各自表示不同的网络状态。

1. `loading`：查询从未在之前运行，请求现在处于待处理状态。即使从高速缓存返回结果，查询仍然具有此网络状态，但是查询也被调度。
2. `setVariables`：如果查询的变量发生变化，网络请求被触发，则网络状态将为`setVariables`，直到该查询的结果返回。当[`options.variables`](＃graphql-query-options-variables)对其查询进行更改时，React用户会看到这一点。
3. `fetchMore`：表示在这个查询中调用了`fetchMore`，所创建的网络请求目前正在运行。
4. `refetch`：这意味着`refetch`在一个查询中被调用，并且refetch请求当前正在运行。
5. 没用。
6. `poll`：表示轮询查询当前正在运行。因此，例如，如果您每10秒钟轮询一次查询，那么当轮询请求发送但未解决时，网络状态将每10秒切换到 `poll`。
7. `ready`：这个查询没有请求正在运行，没有发生错误。一切都好。
8. `error`：此查询没有请求正在运行，但检测到一个或多个错误。

如果网络状态小于7，则相当于[`data.loading`](#graphql-query-data-loading)为真。实际上，您可以用 `data.networkStatus < 7` 替换所有的`data.loading`检查，尽管看不出有什么区别，但建议您使用`data.loading'。

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

Apollo用来从GraphQL端点获取数据的变量。如果要根据用于向服务器发出请求的变量呈现某些信息，此属性将非常有用。

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

强制您的组件重新获取您在 `graphql()` 函数中定义的查询。当您要重新加载组件中的数据时，此方法很有用，或者在错误后重试一次抓取。

`data.refetch` 返回一个承诺，一旦查询执行结束，就会从你的API中获取的新数据解析出来。如果查询失败，承诺将拒绝。

`data.refetch` 函数接受一个 `variables` 对象参数。 `variables`参数将替换HOC（根据你是否指定了一个`query`）选项来查询你所定义的查询`graphql（）`函数。

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

`data.fetchMore` 功能可以让您对查询组件进行分页。要了解有关 `data.fetchMore` 的分页的更多信息，请务必阅读[分页](pagination.html)配方，其中包含有关如何使用React Apollo进行分页的有用插图。

`data.fetchMore`返回一旦解决执行查询更多数据的查询已解决的承诺。

`data.fetchMore`函数使用一个`options`对象参数。 `options`参数可能需要以下属性：

- `[query]`：这是使用`gql` GraphQL标签创建的可选GraphQL文档。如果你指定一个`query`，那么当你调用`data.fetchMore`的时候，那个查询将被提取。如果没有指定`query`，那么将使用`graphql()`HOC的查询。
- `[variables]`：您可能提供的可选变量将与`query`选项或`graphql()`HOC中的查询一起使用（取决于您是否指定了一个`query`）。
- `updateQuery(previousResult, { fetchMoreResult, queryVariables })`：这是您定义的必需的函数，实际上将更新您的分页列表。第一个参数`previousResult`将是您在`graphql()`函数中定义的查询返回的先前数据。第二个参数是一个具有两个属性`fetchMoreResult`和`queryVariables`的对象。 `fetchMoreResult`是新的fetch返回的数据，它使用`data`和`variables`选项从`data.fetchMore`。 `queryVariables`是获取更多数据时使用的变量。使用这些参数，您应该返回一个与您在 `graphql()` 函数中定义的GraphQL查询相同形状的新数据对象。请参阅下面的示例，并确保阅读具有完整示例的[分页](pagination.html) 配方。

**例：**

```js
data.fetchMore({
  updateQuery: (previousResult, { fetchMoreResult, queryVariables }) => {
    return {
      ...previousResult,
      // 将新的Feed数据添加到旧Feed数据的末尾。
      feed: [...previousResult.feed, ...fetchMoreResult.feed],
    },
  },
});
```

<h3 id="graphql-query-data-subscribeToMore">`data.subscribeToMore(options)`</h3>

此功能将设置订阅，每当服务器发送订阅发布时触发更新。这需要在服务器上设置订阅才能正常工作。查看[订阅指南](http://dev.apollodata.com/react/receiving-updates.html#Subscriptions) 和[subscriptions-transport-ws](https://github.com/apollographql/subscriptions-transport-ws) 和[graphql-subscriptions](https://github.com/apollographql/graphql-subscriptions) 获取此设置的更多信息。

该函数返回一个`unsubscribe`函数处理程序，可以在以后取消订阅。

通常的做法是在`componentWillReceiveProps`中包装`subscribeToMore`调用，并在原始查询完成后执行订阅。为确保订阅不会多次创建，您可以将其附加到组件实例。有关详细信息，请参阅示例。

- `[document]`：Document是一个必需的属性，它接受用`graphql-tag`的`gql`模板字符串标签创建的GraphQL订阅。它应该包含一个GraphQL订阅操作，并返回数据。
- `[variables]`：您可能提供的可选变量将与`document`选项一起使用。
- `[updateQuery]`：每次服务器发送更新时运行的可选功能。这将修改HOC查询的结果。第一个参数`previousResult`将是您在`graphql()`函数中定义的查询返回的先前数据。第二个参数是一个具有两个属性的对象。 `subscriptionData`是订阅的结果。 `variables`是与订阅查询一起使用的变量对象。使用这些参数，您应该返回一个与您在`graphql()`函数中定义的GraphQL查询相同形状的新数据对象。这与[`fetchMore`](#graphql-query-data-fetchMore)回调类似。或者，您可以使用[reducer](http://dev.apollodata.com/react/cache-updates.html#resultReducers)作为你的`graphql()`函数的[options](http://dev.apollodata.com/react/queries.html#graphql-options)的一部分来更新查询。
- `[onError]`：一个可选的错误回调。

为了使用订阅结果更新查询的商店，您必须在`graphql()`函数中指定`subscribeToMore` 的 `updateQuery`选项或 `reducer` 选项。

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
          // 使用subscriptionData执行previousResult的更新
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

此函数将设置间隔，并在每次间隔时间间隔发送提取请求。该函数只有一个整数参数，允许您配置希望以毫秒为单位执行查询的频率。换句话说，`interval` 参数表示轮询之间的毫秒数。

轮询是将UI中的数据保存在新鲜空间中的好方法。通过每5000毫秒（例如5秒钟）重新获取数据，您可以有效地模拟实时数据，而无需建立实时后端。

如果您的查询已经在轮询时调用`data.startPolling`，那么当前的轮询过程将被取消，并且以指定的时间间隔开始一个新进程。

您也可以使用[`options.pollInterval`](#graphql-config-options-pollInterval)在您的组件挂载后立即开始轮询。如果您不需要任意地启动和停止轮询，建议您使用[`options.pollInterval`](#graphql-config-options-pollInterval)。

如果你将`interval`设置为0，那么这意味着没有轮询，而不是每个JavaScript事件循环勾选执行一个请求。

**例：**

```js
class MyComponent extends Component {
  componentDidMount() {
    // 在这种具体情况下，您可能需要使用`options.pollInterval`。
    this.props.data.startPolling(1000);
  }

  render() {
    // ...
  }
}

export default graphql(gql`query { ... }`)(MyComponent);
```

<h3 id="graphql-query-data-stopPolling">`data.stopPolling()`</h3>

通过调用此函数，您将停止任何当前的轮询过程。在您调用`data.startPolling`之前，您的查询将不会再次轮询。

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

此功能允许您在任何突变，订阅或提取的上下文之前更新查询的数据。该函数只需要一个单独的参数，这将是另一个函数。参数函数具有以下特征：

```
(previousResult, { variables }) => nextResult
```

第一个参数将是存储中当前存在的查询的数据，并且您将返回具有相同形状的新数据对象。该新数据对象将被写入存储，并且跟踪该数据的任何组件将被反应地更新。

第二个参数是一个具有单个属性`variables`的对象。 `variables` 属性允许你在从store中读取 `previousResult` 时查看正在使用的变量。

此方法将*不*更新服务器上的任何东西。它只会更新客户端缓存中的数据，如果您重新加载JavaScript环境，则更新将消失。

**例：**

```js
data.updateQuery((previousResult) => ({
  ...previousResult,
  count: previousResult.count + 1,
}));
```

<h2 id="graphql-query-options">`config.options`</h2>

返回用于配置如何获取和更新查询的选项对象的对象或函数。

如果`config.options`是一个函数，那么它将以组件的道具为第一个参数。

可用于此对象的选项取决于您作为 `graphql()` 的第一个参数传入的操作类型。以下参考文献将记录您的操作是查询时哪些选项可用。要查看其他可用于不同操作的选项，请参阅[`config.options`](#graphql-config-options)的通用文档。

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

执行查询操作时将使用的变量。这些变量应与查询定义接受的变量相对应。如果你定义`config.options`作为一个函数，那么你可以从你的道具计算你的变量。

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

`fetchPolicy` 是一个选项，可让您指定组件如何与Apollo数据缓存进行交互。默认情况下，您的组件将首先尝试从缓存中读取，如果查询的完整数据位于缓存中，则Apollo将从缓存中简单地返回数据。如果您的查询的完整数据在缓存中为*not*，则Apollo将使用您的网络接口执行您的请求。通过更改此选项，您可以更改此行为。

`fetchPolicy`有效值有：

- `cache-first`：这是我们始终尝试从缓存中读取数据的默认值。如果完成查询所需的所有数据都在缓存中，那么该数据将被返回。如果缓存结果不可用，Apollo将仅从网络中提取。此提取策略旨在最大限度地减少渲染组件时发送的网络请求数。
- `cache-and-network`：这个提取策略将让Apollo首先尝试从缓存中读取数据。如果完成查询所需的所有数据都在缓存中，那么该数据将被返回。但是，无论您的缓存中是否存在完整数据，`fetchPolicy`将*始终*执行与网络接口的查询不同于`cache-first`，只有当查询数据不在缓存中时，才执行查询。该提取策略优化用户获得快速响应，同时还要尝试将高速缓存的数据与服务器数据保持一致，但需要额外的网络请求。
- `network-only`：此提取策略将*永远不会*从缓存中返回初始数据。相反，它将始终使用您的网络接口向服务器发出请求。该提取策略优化了与服务器的数据一致性，但是在可用时立即响应用户的代价。
- `cache-only`：这个抓取策略将*永远不会*使用你的网络接口执行查询。相反，它将始终尝试从缓存中读取。如果查询的数据不存在于缓存中，则会抛出错误。此提取策略允许您只与本地客户端缓存中的数据进行交互，而不会发生任何网络请求，从而保持组件快速，但意味着您的本地数据可能与服务器上的数据不一致。如果您只对Apollo Client缓存中的数据进行交互感兴趣，那么请务必查看您的[`ApolloClient`][]实例中可用的[`readQuery()`][]和[`readFragment()`][]实例。

[`readQuery()`]: ../core/apollo-client-api.html#ApolloClient.readQuery
[`readFragment()`]: ../core/apollo-client-api.html#ApolloClient.readFragment
[`ApolloClient`]: ../core/apollo-client-api.html#apollo-client

**例：**

```js
export default graphql(gql`query { ... }`, {
  options: { fetchPolicy: 'cache-and-network' },
})(MyComponent);
```

<h3 id="graphql-config-options-pollInterval">`options.pollInterval`</h3>

您要开始轮询的间隔（以毫秒为单位）。无论何时经过这个毫秒数，您的查询将使用网络接口执行，另一个执行将使用配置的毫秒数进行调度。

当组件挂载时，此选项将立即开始轮询您的查询。如果要动态启动和停止轮询，则可以使用[`data.startPolling`](#graphql-query-data-startPolling)和[`data.stopPolling`](#graphql-query-data-stopPolling)。

如果将`options.pollInterval`设置为0，那么这意味着每个JavaScript事件循环都不会执行轮询，而不是执行请求。

**例：**

```js
export default graphql(gql`query { ... }`, {
  options: { pollInterval: 5000 },
})(MyComponent);
```

<h3 id="graphql-config-options-notifyOnNetworkStatusChange">`options.notifyOnNetworkStatusChange`</h3>

是否更新网络状态或网络错误会触发组件的重新渲染。

默认值为 `false`。

**例：**

```js
export default graphql(gql`query { ... }`, {
  options: { notifyOnNetworkStatusChange: true },
})(MyComponent);
```
