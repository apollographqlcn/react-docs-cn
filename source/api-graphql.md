---
sidebar_title: "graphql: 容器"
title: "API: graphql 容器"
---

<h2 id="graphql" title="graphql(...)">`graphql(query, [config])(component)`</h2>

```js
import { graphql } from 'react-apollo';
```

`graphql()` 函数是 `react-apollo` 中最重要的部分。使用此函数，您可以创建基于 Apollo store 中的数据来反应性地执行查询和更新的高阶组件。`graphql()` 函数返回一个函数，它将“增强”任何具有反应性 GraphQL 功能的组件。这与 [`react-redux` 的 `connect`][] 函数使用的 React [高阶组件][]模式一致。

[高阶组件]: https://facebook.github.io/react/docs/higher-order-components.html
[`react-redux` 的 `connect`]: https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options

`graphql()` 函数可以这样使用：

```js
function TodoApp({ data: { todos } }) {
  return (
    <ul>
      {todos.map(({ id, text }) => (
        <li key={id}>{text}</li>
      ))}
    </ul>
  );
}

export default graphql(gql`
  query TodoAppQuery {
    todos {
      id
      text
    }
  }
`)(TodoApp);
```

您还可以定义一个中间函数，使用 `graphql()` 函数将组件联接起来，如下所示：

```js
// 创建我们的增强器函数。
const withTodoAppQuery = graphql(gql`query { ... }`);

// 增强我们的组件。
const TodoAppWithData = withTodoAppQuery(TodoApp);

// 导出增强组件。
export default TodoAppWithData;
```

或者，您还可以在 React 类组件上使用 `graphql()` 函数作为[装饰器][]。

[装饰器]: https://github.com/wycats/javascript-decorators

如果是这样，你的代码可能如下所示：

```js
@graphql(gql`
  query TodoAppQuery {
    todos {
      id
      text
    }
  }
`)
export default class TodoApp extends Component {
  render() {
    const { data: { todos } } = this.props;
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

如果您的组件树根部有一个 [`<ApolloProvider/>`](#ApolloProvider) 组件提供一个 [`ApolloClient`][] 实例用于获取数据，`graphql()` 函数将只能提供对 GraphQL 数据的访问。

[`ApolloClient`]: http://dev.apollodata.com/core/apollo-client-api.html#apollo-client

根据 GraphQL 的操作是一个[查询](#queries)，还是一个[突变](#mutations)，或是一个[订阅](#subscriptions)，您使用 `graphql()` 函数增强的组件的行为会有所不同。有关每种类型的功能和可用选项的更多信息，请参阅相应的API文档。

在我们研究每种操作的具体行为之前，让我们先看看 `config` 对象。


<h2 id="graphql-config">`config`</h2>

在你的 GraphQL 文本之后，`config` 对象是你传入 `graphql()` 函数的第二个参数。该配置是可选的，并允许您向高阶组件添加一些自定义行为。

```js
export default graphql(
  gql`{ ... }`,
  config, // <- `config` 对象.
)(MyComponent);
```

让我们来看看你的 `config` 对象所有可能的属性。

<h3 id="graphql-config-options">`config.options`</h3>

`config.options` 是一个对象或函数，它允许您定义组件在处理 GraphQL 数据时应使用的具体行为。

可用于配置的具体选项取决于您为 `graphql()` 传递的第一个参数 -- 操作类型。针对[查询](api-queries.html#graphql-query-options)和[突变](api-mutations.html#graphql-mutation-options)，有特定的选项可以配置。

您可以将 `config.options` 定义为普通对象，或者您可以从一个函数中计算您的选项，该函数接收组件的 props 作为参数。

**例：**

```js
export default graphql(gql`{ ... }`, {
  options: {
    // 在这里填写配置。
  },
})(MyComponent);
```

```js
export default graphql(gql`{ ... }`, {
  options: (props) => ({
    // 选项从这里的 `props` 计算而得。
  }),
})(MyComponent);
```

<h3 id="graphql-config-props">`config.props`</h3>

`config.props` 属性允许你定义一个 map 函数，它包含你的所有 props，包括由 `graphql()` 函数（用于查询的 [`props.data`](#graphql-query-data)，用于突变的 [`props.mutate`](#graphql-mutation-mutate)）添加的属性，并且允许您计算一个新的 props 对象，提供给`graphql()` 正在包装的组件。

您定义的函数几乎与 [Recompose 中的 `mapProps`][] 完全一样，提供了相同的便利，并且不需要引入一个新的库。

[Recompose 中的 `mapProps`]: https://github.com/acdlite/recompose/blob/2e71fdf4270cc8022a6574aaf00731bfc25dcae6/docs/API.md#mapprops

如果你想将复杂的函数调用抽象成一个简单的，可以传递给你的组件的 prop，那么 `config.props` 是最有用的。

`config.props` 的另一个好处，是它也允许你将你的纯 UI 组件与 GraphQL 和 Apollo 的关系分离开来。您可以将纯 UI 组件写在一个文件里，然后保留所需的逻辑，以便它们在项目中别的地方与 store 进行交互。您可以通过只需要渲染所需的 props 的纯 UI 组件实现这个，`config.props` 可以包含从 GraphQL API 提供的数据中完全提供你的纯组件所需的 props 的逻辑。

**例：**

此示例使用了 [`props.data.fetchMore`](#graphql-query-data-fetchMore)。

```js
export default graphql(gql`{ ... }`, {
  props: ({ data: { fetchMore } }) => ({
    onLoadMore: () => {
      fetchMore({ ... });
    },
  }),
})(MyComponent);

function MyComponent({ onLoadMore }) {
  return (
    <button onClick={onLoadMore}>
      Load More!
    </button>
  );
}
```

<h3 id="graphql-config-skip">`config.skip`</h3>

如果 `config.skip` 值为真，那么所有的 React Apollo 代码将被*完全*跳过。就好像`graphql()`函数是一个简单的标识函数。您的组件将会像 `graphql()` 函数不存在那般。

除了将布尔值传递给 `config.skip`，您也可以将函数传递给 `config.skip`。该函数将接收您的组件 props，并返回一个布尔值。如果布尔值返回真，则跳过行为将生效。

如果你想要基于某些 prop 使用不同的查询，`config.skip` 将会十分有用。您可以在下面的示例中看到这一点。

**例：**

```js
export default graphql(gql`{ ... }`, {
  skip: props => !!props.skip,
})(MyComponent);
```

以下示例使用 [`compose()`](#compose) 函数同时使用多个 `graphql()` 增强器。

```js
export default compose(
  graphql(gql`query MyQuery1 { ... }`, { skip: props => !props.useQuery1 }),
  graphql(gql`query MyQuery2 { ... }`, { skip: props => props.useQuery1 }),
)(MyComponent);

function MyComponent({ data }) {
  // 数据可能来自 `MyQuery1` 或 `MyQuery2`，具体取决于 `useQuery1` prop 的值。
  console.log(data);
}
```

<h3 id="graphql-config-name">`config.name`</h3>

该属性允许您配置传递给组件的 prop 的名称。默认情况下，如果您传递给 `graphql()` 的 GraphQL 文本是一个查询，那么您的 prop 将被命名为 [`data`](#graphql-query-data)。如果你传递的是一个突变，那么你的 prop 将被命名为 [`mutate`](#graphql-mutation-mutate)。当您尝试为同一个组件的使用多个查询或突变时，这些默认名称会相互冲突。为了避免冲突，您可以使用 `config.name` 为每个查询或突变高阶组件提供一个新的名字给 prop。

**例：**

此示例使用 [`compose`](#compose) 函数将多个 `graphql()` 高阶组件组合在一起。

```js
export default compose(
  graphql(gql`mutation (...) { ... }`, { name: 'createTodo' }),
  graphql(gql`mutation (...) { ... }`, { name: 'updateTodo' }),
  graphql(gql`mutation (...) { ... }`, { name: 'deleteTodo' }),
)(MyComponent);

function MyComponent(props) {
  // 我们有三个不同的名称，而不是默认的 prop 名称 `mutate`。
  console.log(props.createTodo);
  console.log(props.updateTodo);
  console.log(props.deleteTodo);

  return null;
}
```

<h3 id="graphql-config-withRef">`config.withRef`</h3>

通过将 `config.withRef` 设置为真，您可以使用高阶 GraphQL 组件实例上的 `getWrappedInstance` 方法从该高阶 GraphQL 组件实例获取包裹组件。

当您想要调用在包裹组件的类实例上定义的函数或访问其属性时，可能需要将其设置为真。

下面是一个这种情况下的例子。

**例：**

此示例使用 [React `ref` 功能][]。

[React `ref` 功能]: https://facebook.github.io/react/docs/refs-and-the-dom.html

```js
class MyComponent extends Component {
  saySomething() {
    console.log('Hello, world!');
  }

  render() {
    // ...
  }
}

const MyGraphQLComponent = graphql(
  gql`{ ... }`,
  { withRef: true },
)(MyComponent);

class MyContainerComponent extends Component {
  render() {
    return (
      <MyGraphQLComponent
        ref={component => {
          assert(component.getWrappedInstance() instanceof MyComponent);
          // 我们可以在组件类实例上调用方法。
          component.saySomething();
        }}
      />
    );
  }
}
```

<h3 id="graphql-config-alias">`config.alias`</h3>

默认情况下，React Apollo 组件的显示名称为 `Apollo(${WrappedComponent.displayName})`。这是大多数使用高阶组件的 React 库所使用的模式。但是，当您使用多个高阶组件时查看 [React Devtools][]，可能会有点困惑。

[React 开发者工具]: https://camo.githubusercontent.com/42385f70ef638c48310ce01a675ceceb4d4b84a9/68747470733a2f2f64337676366c703535716a6171632e636c6f756466726f6e742e6e65742f6974656d732f30543361333532443366325330423049314e31662f53637265656e25323053686f74253230323031372d30312d3132253230617425323031362e33372e30302e706e673f582d436c6f75644170702d56697369746f722d49643d626536623231313261633434616130636135386432623562616265373336323626763d3236623964363434

如果要配置高阶组件包装器的名称，可以使用 `config.alias` 属性。因此，假设你将 `config.alias` 设置为 `'withCurrentUser'`，你的包装器组件显示名称将是 `withCurrentUser(${WrappedComponent.displayName})`，而不是 `Apollo(${WrappedComponent.displayName})`。

**例：**

此示例使用 [`compose`](#compose) 函数将多个 `graphql()` 高阶组件组合在一起。

```js
export default compose(
  graphql(gql`{ ... }`, { alias: 'withCurrentUser' }),
  graphql(gql`{ ... }`, { alias: 'withList' }),
)(MyComponent);
```

<h2 id="compose" title="compose(...)">`compose(...enhancers)(component)`</h2>

```js
import { compose } from 'react-apollo';
```

为了实用目的，`react-apollo` 导出一个 `compose` 函数。使用此函数，您可以利落地一次使用多个组件增强器。包括多个 [`graphql()`](#graphql)，[`withApollo()`](#withApollo) 或 [Redux `connect()`][] 增强器。当您使用多个增强器时，这将会理清您的代码。 [Redux][] 也导出一个 `compose` 函数，[Recompose][] 也是如此，所以你可以有选择地从合适的库中使用这个函数。

一个重要的注意事项是，`compose()` _首先_执行最后一个增强器，并根据增强器列表依次向后执行。为了说明这种情况，我们调用三个函数：`funcC(funcB(funcA(component)))` 相当于这样调用 `compose()`：`compose(funcC, funcB, funcA)(component)`。如果还不明白，你可以想象 [Lodash 中的`flowRight()`][] 函数，它们具有相同的行为。

[Redux `connect()`]: https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options
[Redux]: http://redux.js.org/
[Recompose]: https://github.com/acdlite/recompose
[Lodash 中的 `flowRight()`]: https://lodash.com/docs/4.17.4#flowRight

**例：**

```js
export default compose(
  withApollo,
  graphql(`query { ... }`),
  graphql(`mutation { ... }`),
  connect(...),
)(MyComponent);
```
