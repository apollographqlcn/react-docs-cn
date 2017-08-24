---
sidebar_title: "graphql: 突变"
title: "API: 带突变的graphql容器"
---

> 这篇文章特别讲解 `graphql()` 高阶组件使如何用突变。要查看适用于所有操作的选项，请参阅 [graphql 容器 API 文档](api-graphql.html)。

你传递给 `graphql()` 函数的操作决定了组件的行为方式。如果你将突变传递到你的 `graphql()` 函数中，那么 Apollo 将在你的组件 props 中设置一个可以任意调用的 `mutate` 函数。

如下是一个使用 `graphql()` 突变的示例组件：

```js
export default graphql(gql`
  mutation TodoCompleteMutation($id: ID!) {
    completeTodo(id: $id) {
      id
      text
      completed
    }
  }
`)(TodoCompleteButton);

function TodoCompleteButton({ todoID, mutate }) {
  return (
    <button onClick={() => mutate({ variables: { id: todoID } })}>
      Complete
    </button>
  );
}
```

为了更好地理解本文所述的使用 `graphql()` 函数执行突变，请务必先阅读[突变文档](mutations.html)。下文是有关用于突变的 `graphql()` 函数支持的所有功能的技术概述。

<h2 id="graphql-mutation-mutate">`props.mutate`</h2>

将突变传递给 `graphql()` 时创建的高阶组件，将为你的组件提供一个名为 `mutate` 的单个 prop。与 `graphql()` 传递查询时所得到的 `data` 属性不同，mutate 是一个函数。

`mutate` 函数实际上将使用网络接口来执行你的突变，从而改变你的数据。`mutate` 函数还将以你定义的方式更新缓存。

要了解更多有关突变如何工作的信息，请务必查阅[突变使用文档](mutations.html)。

`mutate` 函数接受与 [突变的 `config.options`](#graphql-mutation-options) 相同的选项，所以要确保阅读本文档，以了解可以传入 `mutate` 函数的内容。

`mutate`函数接受相同选项的原因是它将_默认_使用 [`config.options`](#graphql-mutation-options) 中的选项。当你将对象传递到 `mutate` 函数中时，你只需覆盖 [`config.options`](#graphql-mutation-options) 中已有的内容。

**例：**

```js
function MyComponent({ mutate }) {
  return (
    <button onClick={() => {
      mutate({
        variables: { foo: 42 },
      });
    }}>
      Mutate
    </button>
  );
}

export default graphql(gql`mutation { ... }`)(MyComponent);
```

<h2 id="graphql-mutation-options">`config.options`</h2>

该选项是一个对象或函数，返回用于配置如何请求和更新查询的选项对象。

如果 `config.options` 是一个函数，那么它将以组件的 props 为第一个参数。

可用于此对象的选项取决于你为 `graphql()` 的第一个参数传入的操作类型。以下参考文档将记录你的操作是突变时哪些选项可用。要查看其他用于不同操作的选项，请参阅 [`config.options`](api-graphql.html#graphql-config-options) 的通用文档。

此选项对象中接受的属性也可以被 [`props.mutate`](#graphql-mutation-mutate) 函数接受。传入 `mutate` 函数的任何选项将优先于 `config` 对象中定义的选项。

**例：**

```js
export default graphql(gql`mutation { ... }`, {
  options: {
    // 这里填写选项。
  },
})(MyComponent);
```

```js
export default graphql(gql`mutation { ... }`, {
  options: (props) => ({
    // 这里根据 props 计算选项。
  }),
})(MyComponent);
```

```js
function MyComponent({ mutate }) {
  return (
    <button onClick={() => {
      mutate({
        // 这里的选项来源于组件的 `props` 和 state。
      });
    }}>
      Mutate
    </button>
  )
}

export default graphql(gql`mutation { ... }`)(MyComponent);
```

<h3 id="graphql-mutation-options-variables">`options.variables`</h3>

该选项是用于执行突变操作的变量。这些变量应该与你的突变定义接受的变量相对应。如果将 `config.options` 定义为函数，或者将变量传递到 [`props.mutate`](#graphql-mutation-mutate) 函数中，那么可以从组件的 props 和 state 中计算变量。

**例：**

```js
export default graphql(gql`
  mutation ($foo: String!, $bar: String!) {
    ...
  }
`, {
  options: (props) => ({
    variables: {
      foo: props.foo,
      bar: props.bar,
    },
  }),
})(MyComponent);
```

<h3 id="graphql-mutation-options-optimisticResponse">`options.optimisticResponse`</h3>

通常情况下，当你执行突变更改数据时，很容易在请求服务器之前提前预测到，突变的响应是什么。乐观响应选项可让你通过在突变实际完成之前模拟 UI 中突变的结果，使你的突变响应更快。

要了解更多关于乐观数据的好处以及如何使用它，请务必阅读 [Optimistic UI](optimistic-ui.html) 文档。

这个乐观响应将与 [`options.update`](#graphql-mutation-options-update) 和 [`options.updateQueries`](#graphql-mutation-options-updateQueries) 一起使用，以将更新应用于缓存中，在应用实际响应的更新之前缓存将会回滚。

**例：**

```js
function MyComponent({ newText, mutate }) {
  return (
    <button onClick={() => {
      mutate({
        variables: {
          text: newText,
        },
        // 乐观响应包含下面的 GraphQL 突变文档中包含的所有字段。
        optimisticResponse: {
          createTodo: {
            id: -1, // 一个临时id。服务器返回真实的ID。
            text: newText,
            completed: false,
          },
        },
      });
    }}>
      Add Todo
    </button>
  );
}

export default graphql(gql`
  mutation ($text: String!) {
    createTodo(text: $text) {
      id
      text
      completed
    }
  }
`)(MyComponent);
```

<h3 id="graphql-mutation-options-update">`options.update`</h3>

此选项允许你根据突变的结果更新你的 store。默认情况下，Apollo 客户端将更新你的 store 中的所有重叠节点。与你定义的 `dataIdFromObject` 返回的 ID 相同的任何内容将使用你的突变结果的新字段进行更新。但是，有时这还不够。有时你可能希望根据缓存中当前的数据以指定方式更新缓存。对于这些更新，你可以使用 `options.update` 函数。

`options.update` 需要两个参数。第一个是 [`DataProxy`][] 对象的一个​​实例，它有一些方法可以让你与 store 中的数据进行交互。第二个是你的突变的响应 - 乐观响应或服务器返回的实际响应。

为了更改你的 store 中的数据，需要调用 [`DataProxy`][] 实例上的方法，如 [`writeQuery`][] 和 [`writeFragment`][]。这将更新你的缓存并重新渲染任何数据受查询影响的 GraphQL 组件。

要读取你正在更改的 store 中的数据，请确保使用 [`DataProxy`][] 的诸如 [`readQuery`][] 和 [`readFragment`][] 方法。

有关更多使用 `options.update` 函数执行突变之后更新缓存的信息，请务必阅读 [Apollo 客户端技术文档相关的主题](http://dev.apollodata.com/core/read-and-write.html#updating-the-cache-after-a-mutation)。

[`DataProxy`]: http://dev.apollodata.com/core/apollo-client-api.html#DataProxy
[`writeQuery`]: http://dev.apollodata.com/core/apollo-client-api.html#DataProxy.writeQuery
[`writeFragment`]: http://dev.apollodata.com/core/apollo-client-api.html#DataProxy.writeFragment
[`readQuery`]: http://dev.apollodata.com/core/apollo-client-api.html#DataProxy.readQuery
[`readFragment`]: http://dev.apollodata.com/core/apollo-client-api.html#DataProxy.readFragment

**例：**

```js
const query = gql`{ todos { ... } }`

export default graphql(gql`
  mutation ($text: String!) {
    createTodo(text: $text) { ... }
  }
`, {
  options: {
    update: (proxy, { data: { createTodo } }) => {
      const data = proxy.readQuery({ query });
      data.todos.push(createTodo);
      proxy.writeQuery({ query, data });
    },
  },
})(MyComponent);
```

<h3 id="graphql-mutation-options-refetchQueries">`options.refetchQueries`</h3>

有时，当你进执行突变时，你还需要更新查询中的数据，以便用户可以看到最新的用户界面。有很多细微的方式来更新缓存中的数据，包括 [`options.updateQueries`](#graphql-mutation-options-updateQueries) 和 [`options.update`](#graphql-mutation-options-update)。但是，你可以通过使用 `options.refetchQueries` 牺牲一些效率来更可靠地更新缓存中的数据。

`options.refetchQueries` 将使用你的网络接口执行一个或多个查询，然后将这些查询的结果范式化到缓存中。该选项允许你能重新获取你之前已获取的查询，或者获取全新的查询。

`options.refetchQueries` 是一个字符串或对象的数组。

如果 `options.refetchQueries` 是一个字符串数组，那么 Apollo 客户端将寻找与提供的字符串同名的任意查询，并使用它们当前的变量来获取这些查询。因此，例如，如果你有一个查询名为 `Comments` 的 GraphQL 查询组件（该查询可能是：`query Comments {...}`），并且将包含 `Comments` 的字符串数组传递给 `options.refetchQueries`，然后 `Comments` 查询将被重新执行，当它解析完时最新的数据将会自动更新在你的UI中。

如果 `options.refetchQueries` 是一个对象的数组，那么对象必须有两个属性：

- `query`: Query 是一个必需的属性，它接受使用 `graphql-tag` 的 `gql` 模板字符串标签创建的 GraphQL 查询。它应该包含一个单独的 GraphQL 查询操作，一旦突变完成，它将被执行。
- `[variables]`: 当 `query` 接受一些变量时，这是一个可选的变量对象。

如果指定了具有此形状的对象数组，那么 Apollo 客户端将使用其中的变量来重新获取这些查询。

**例：**

```js
export default graphql(gql`mutation { ... }`, {
  options: {
    refetchQueries: [
      'CommentList',
      'PostList',
    ],
  },
})(MyComponent);
```

```js
import { COMMENT_LIST_QUERY } from '../components/CommentList';

export default graphql(gql`mutation { ... }`, {
  options: (props) => ({
    refetchQueries: [
      {
        query: COMMENT_LIST_QUERY,
      },
      {
        query: gql`
          query ($id: ID!) {
            post(id: $id) {
              commentCount
            }
          }
        `,
        variables: {
          id: props.postID,
        },
      },
    ],
  }),
})(MyComponent);
```

<h3 id="graphql-mutation-options-updateQueries">`options.updateQueries`</h3>

**注意：我们建议使用 [update](#graphql-mutation-options-update) 而不是 `updateQueries`。**

默认情况下，Apollo 客户端将更新你的 store 中的所有重叠节点。与你定义的 `dataIdFromObject` 返回的 ID 相同的任何内容将使用你的突变结果的新字段进行更新。但是，有时这还不够。有时你可能希望根据缓存中当前的数据以指定方式更新缓存。对于这些更新，你可以使用 `options.updateQueries` 函数。

`options.updateQueries` 接收一个对象，其中查询名称是键，而 reducer 函数是值。如果你熟悉Redux，定义 `options.updateQueries` reducers 与定义 Redux reducer 非常相似。该对象看起来像这样：

```js
{
  Comments: (previousData, { mutationResult, queryVariables }) => nextData,
}
```

请确保你的 `options.updateQueries` 对象的键与你在应用中实际执行过的查询相对应。查询名称将是在指定 `query` 操作类型后放置的名称。因此，例如在以下查询中：

```graphql
query Comments {
  entry(id: 5) {
    comments {
      ...
    }
  }
}
```

查询名称将为 `Comments`。如果在你的应用中某处尚未执行一个名为 `Comments` 的 GraphQL 查询，那么 Apollo 将永远不会运行此 reducer 函数，`options.updateQueries` 中的键/值对将被忽略。

你提供给该函数的第一个参数，是你的查询的旧数据对象的值。所以如果你的键是 `Comments`，那么第一个参数将是你的 `Comments` 查询返回的最后一个数据对象，或者是使用 `Comments` 查询的任意组件所使用的当前对象。

函数的第二个参数值是具有三个属性的对象：

- `mutationResult`：`mutationResult` 属性将表示你请求服务器后的突变结果。如果你提供了一个 [`options.optimisticResponse`](#graphql-mutation-options-optimisticResponse)，那么 `mutationResult` 可能就是那个对象。
- `queryVariables`：执行查询的最后一组变量。这是有用的，因为当你指定查询名称时，它将只更新当前变量集相应的 store 中的数据。
- `queryName`：这是你正在更新的查询的名称。它与你为 `options.updateQueries` 提供的键同名。

你的 `options.updateQueries` 函数的返回值_必须_与你的第一个 `previousData` 参数的形状相同。但是，你_绝不能_更改 `previousData` 对象。相反，你必须根据变更创建一个新对象。就像 Redux reducer。

要了解更多有关 `options.updateQueries` 的信息，请阅读我们的关于[使用 `updateQueries` 控制 Store](cache-updates.html#updateQueries) 的文档。

**例：**

```js
export default graphql(gql`
  mutation ($text: String!) {
    submitComment(text: $text) { ... }
  }
`, {
  options: {
    updateQueries: {
      Comments: (previousData, { mutationResult }) => {
        const newComment = mutationResult.data.submitComment;
        // 注意这里我们将返回一个新的 `previousData` 副本，而不是改变它。这就像一个 Redux reducer！
        return {
          ...previousData,
          entry: {
            ...previousData.entry,
            comments: [newComment, ...previousData.entry.comments],
          },
        };
      },
    },
  },
})(MyComponent);
```
