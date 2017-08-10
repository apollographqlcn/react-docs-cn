---
sidebar_title: "graphql: 容器"
title: "API: graphql 容器"
---

<h2 id="graphql" title="graphql(...)">`graphql(query, [config])(component)`</h2>

```js
import { graphql } from 'react-apollo';
```

`graphql()`函数是`react-apollo`导出的最重要的东西。使用此功能，你可以创建可以基于Apollo store中的数据来执行查询和反应更新的高阶组件。 `graphql()`函数返回一个函数，它将“增强”任何具有反应性GraphQL功能的组件。这就是[`react-redux`’s `connect`][]函数使用的React [高阶分量] []模式。

[高阶组件]: https://facebook.github.io/react/docs/higher-order-components.html
[`react-redux` 的 `connect`]: https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options

`graphql()`函数可以这样使用：

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

你还可以定义一个中间函数，并使用`graphql()`函数挂接组件，如下所示：

```js
// 创建我们的增强器函数。
const withTodoAppQuery = graphql(gql`query { ... }`);

// 增强我们的组件。
const TodoAppWithData = withTodoAppQuery(TodoApp);

// 导出增强的组件。
export default TodoAppWithData;
```

或者，你还可以在React类组件上使用`graphql()`函数作为[装饰器][]。

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

如果你的组件树中有一个[`<ApolloProvider/>`](#ApolloProvider)组件高于[`ApolloClient`][]，`graphql()`函数将只能提供对GraphQL数据的访问。将用于获取数据的实例。

[`ApolloClient`]: ../core/apollo-client-api.html#apollo-client

根据GraphQL操作是一个[查询](#queries)，一个[突变](#mutations)或一个[订阅](#subscriptions)，你使用`graphql()`函数增强的组件的行为会有所不同）。有关每种类型的功能和可用选项的更多信息，请参阅相应的API文档。

在我们研究每个操作的具体行为之前，让我们看看`config`对象。


<h2 id="graphql-config">`config`</h2>

在你的GraphQL文件之后，`config`对象是你传入`graphql()`函数的第二个参数。该配置是可选的，并允许你向高阶组件添加一些自定义行为。

```js
export default graphql(
  gql`{ ... }`,
  config, // <- `config` 对象.
)(MyComponent);
```

让我们来看看你的`config`对象的所有属性。

<h3 id="graphql-config-options">`config.options`</h3>

`config.options` 是一个对象或函数，它允许你定义组件在处理GraphQL数据时应使用的具体行为。

可用于配置的具体选项取决于你作为`graphql()`的第一个参数传递的操作。有[查询](api-queries.html#graphql-query-options)和[mutations](api-mutations.html#graphql-mutation-options)的特定选项。

你可以将`config.options`定义为普通对象，或者你可以从将组件道具视为参数的函数中计算你的选项。

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
    // 选项从`props`在这里计算。
  }),
})(MyComponent);
```

<h3 id="graphql-config-props">`config.props`</h3>

`config.props`属性允许你定义一个map函数，它包含你的道具，包括由`graphql()`函数（[`props.data`](#graphql-query-data)添加的道具，用于查询和[`props.mutate`](#graphql-mutation-mutate) 用于突变），并允许你计算将提供给`graphql()`正在包装的组件的新道具对象。

你定义的函数几乎完全像[`mapProps` from Recompose][]提供了相同的优点，而不需要另一个库。

[Recompose 中的 `mapProps`]: https://github.com/acdlite/recompose/blob/2e71fdf4270cc8022a6574aaf00731bfc25dcae6/docs/API.md#mapprops

当你想将复杂的函数调用抽象成一个简单的代码，你可以传递给你的组件，`config.props`是最有用的。

`config.props`的另一个好处是它也允许你将你的纯UI组件与GraphQL和Apollo的关系分离开来。你可以将纯UI组件写入一个文件，然后保留所需的逻辑，以便它们与项目中完全不同的位置与商店进行交互。你可以通过纯粹的UI组件完成此任务，只需要渲染所需的道具，`config.props` 可以包含从GraphQL API提供的数据完全提供纯组件所需的道具的逻辑。

**例：**

此示例使用[`props.data.fetchMore`](#graphql-query-data-fetchMore)。

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

如果`config.skip`是true，那么所有的React Apollo代码将被*完全*跳过。就好像`graphql()`函数是一个简单的身份函数。你的组件将会像`graphql()`函数那样不起作用。

而不是将布尔值传递给`config.skip`，你也可以将函数传递给`config.skip`。该函数将占用你的组件道具，并返回一个布尔值。如果布尔值返回true，则跳过行为将生效。

`config.skip`是特别有用的，如果你想使用一个不同的查询基于一些道具。你可以在下面的示例中看到这一点。

**例：**

```js
export default graphql(gql`{ ... }`, {
  skip: props => !!props.skip,
})(MyComponent);
```

以下示例使用[`compose()`](#compose)函数同时使用多个 `graphql()` 增强器。

```js
export default compose(
  graphql(gql`query MyQuery1 { ... }`, { skip: props => !props.useQuery1 }),
  graphql(gql`query MyQuery2 { ... }`, { skip: props => props.useQuery1 }),
)(MyComponent);

function MyComponent({ data }) {
  // 数据可能来自“`MyQuery1`或`MyQuery2`，具体取决于`useQuery1` 属性的值。
  console.log(data);
}
```

<h3 id="graphql-config-name">`config.name`</h3>

该属性允许你配置传递给组件的prop的名称。默认情况下，如果你传递给`graphql()`的GraphQL文档是一个查询，那么你的prop将被命名为[`data`](#graphql-query-data)。如果你传递一个突变，那么你的prop将被命名为[`mutate`](#graphql-mutation-mutate)。当你尝试使用相同组件的多个查询或突变时，适当的这些默认名称会相冲突。为了避免冲突，你可以使用`config.name`从每个查询中提供prop或者HOC一个新的名字。

**例：**

此示例使用[`compose`](#compose)函数将多个`graphql()`HOC组合在一起。

```js
export default compose(
  graphql(gql`mutation (...) { ... }`, { name: 'createTodo' }),
  graphql(gql`mutation (...) { ... }`, { name: 'updateTodo' }),
  graphql(gql`mutation (...) { ... }`, { name: 'deleteTodo' }),
)(MyComponent);

function MyComponent(props) {
  // 而不是默认的prop名称`mutate`， 我们有三个不同的名称。
  console.log(props.createTodo);
  console.log(props.updateTodo);
  console.log(props.deleteTodo);

  return null;
}
```

<h3 id="graphql-config-withRef">`config.withRef`</h3>

通过将`config.withRef`设置为true，你可以使用高阶GraphQL组件实例上的 `getWrappedInstance` 方法从高阶GraphQL组件获取包裹组件的实例。

当你想要调用函数或访问在包裹组件的类实例上定义的属性时，可能需要将其设置为true。

下面你可以看到一个这个行为的例子。

**例：**

此示例使用[React`ref`功能][]。

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

默认情况下，React Apollo组件的显示名称为`Apollo(${WrappedComponent.displayName})`。这是大多数使用更高阶组件的React库使用的模式。但是，当你使用多个更高级别的组件并且你查看[React Devtools][]时，可能会有点混乱。

[React 开发者工具]: https://camo.githubusercontent.com/42385f70ef638c48310ce01a675ceceb4d4b84a9/68747470733a2f2f64337676366c703535716a6171632e636c6f756466726f6e742e6e65742f6974656d732f30543361333532443366325330423049314e31662f53637265656e25323053686f74253230323031372d30312d3132253230617425323031362e33372e30302e706e673f582d436c6f75644170702d56697369746f722d49643d626536623231313261633434616130636135386432623562616265373336323626763d3236623964363434

要配置高阶组件包装器的名称，可以使用`config.alias`属性。所以例如，如果你将`config.alias`设置为`'withCurrentUser'`，你的包装器组件显示名称将是`withCurrentUser(${WrappedComponent.displayName})`而不是 `Apollo(${WrappedComponent.displayName})`。

**例：**

此示例使用[`compose`](#compose) 函数将多个`graphql()`HOC组合在一起。

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

为了实用目的，`react-apollo` 导出一个`compose`函数。使用此功能，你可以一次干净地使用几个组件增强器。包括多个[`graphql()`](#graphql)，[`withApollo()`](#withApollo)或[Redux `connect()`][]增强器。当你使用多个增强器时，这应该会清理你的代码。 [Redux][]还导出一个`compose`函数，[Recompose][]也是如此，所以你可以选择使用这个函数，无论哪个库最适合。

一个重要的注意事项是，`compose()` 执行最后一个增强器_first_，并通过增强器列表向后运行。为了说明这样调用三个函数：`funcC(funcB(funcA(component)))` 相当于这样调用`compose()`：`compose(funcC, funcB, funcA)(component)`。如果这样做没有意义，你可以考虑使用[Lodash 中的`flowRight()`][]，否则它们具有相同的行为。

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
