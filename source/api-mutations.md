---
sidebar_title: "graphql: 突变"
title: "API: 具有突变的graphql容器"
---

> 这篇文章特别关于使用 `graphql()` 更高阶组件的突变。要查看适用于所有操作的选项，请参阅[一般graphql容器API文档]（/ reactions / api-graphql.html）。

您传递到 `graphql()` 函数的操作决定了组件的行为方式。如果您将变量传递到您的 `graphql()` 函数中，那么Apollo将在您的组件道具中随时设置一个`mutate` 函数。

这是一个使用 `graphql()` 函数的变量的示例组件：

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

为了更自然地使用`graphql()`函数的突变概述，请务必阅读[突变文档文章](mutations.html)。有关用于突变的 `graphql()` 函数支持的所有功能的技术概述，请继续。

<h2 id="graphql-mutation-mutate">`props.mutate`</h2>

当将变量传递给 `graphql()` 时创建的高阶组件将为您的组件提供一个名为 `mutate` 的单个参考。与`graphql（）`传递查询时所得到的`data` 属性不同，mutate是一个函数。

`mutate` 函数实际上将使用网络接口来执行您的突变，从而突变您的数据。`mutate` 函数还将以您定义的方式更新缓存。

要了解有关突变如何工作的更多信息，请务必查看[突变使用文档](mutations.html)。

`mutate`函数接受与[`config.options` for mutations](#graphql-mutation-options)相同的选项，所以要确保阅读文档，以了解可以传入 `mutate` 函数的内容。

`mutate`函数接受相同选项的原因是它将使用[`config.options`](#graphql-mutation-options) _by default_中的选项。当您将对象传递到`mutate` 函数中时，您只需覆盖[`config.options`](#graphql-mutation-options) 中已有的内容。

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

返回用于配置如何获取和更新查询的选项对象的对象或函数。

如果`config.options`是一个函数，那么它将以组件的属性为第一个参数。

可用于此对象的选项取决于您作为`graphql()`的第一个参数传入的操作类型。以下参考文献将记录当您的手术发生突变时哪些选项可用。要查看其他可用于不同操作的选项，请参阅[`config.options`](#graphql-config-options)的通用文档。

此选项对象中接受的属性也可以被[`props.mutate`](#graphql-mutation-mutate) 函数接受。传入 `mutate` 函数的任何选项将优先于 `config` 对象中定义的选项。

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
    // 这里根据props计算选项。
  }),
})(MyComponent);
```

```js
function MyComponent({ mutate }) {
  return (
    <button onClick={() => {
      mutate({
        // 选项是这里的`props`和组件状态的组件。
      });
    }}>
      Mutate
    </button>
  )
}

export default graphql(gql`mutation { ... }`)(MyComponent);
```

<h3 id="graphql-mutation-options-variables">`options.variables`</h3>

用于执行变异操作的变量。这些变量应该与您的突变定义接受的变量相对应。如果将`config.options`定义为函数，或者将变量传递到[`props.mutate`](#graphql-mutation-mutate) 函数中，那么可以从道具和组件状态计算变量。

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

通常，当您突变数据时，很容易就可以预测在询问服务器之前，突变的响应是什么。乐观的回应选项可让您通过在突变实际完成之前模拟UI中突变的结果，使您的突变变得更快。

要了解更多关于乐观数据的好处以及如何使用它，请务必阅读[Optimistic UI](optimistic-ui.html) 上的文章。

这个乐观的响应将与[`options.update`](#graphql-mutation-options-update) 和[`options.updateQueries`](#graphql-mutation-options-updateQueries) 一起使用，以将更新应用于缓存中在应用实际响应的更新之前将会回滚。

**例：**

```js
function MyComponent({ newText, mutate }) {
  return (
    <button onClick={() => {
      mutate({
        variables: {
          text: newText,
        },
        // 乐观的响应包含下面的GraphQL突变文档中包含的所有字段。
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

此选项允许您根据突变的结果更新您的Store。默认情况下，Apollo Client将更新您Store中的所有重叠节点。与您定义的 `dataIdFromObject` 返回的共享ID相同的任何内容将使用您的突变结果的新字段进行更新。但是，有时这个还不够。有时您可能希望以取决于缓存中当前数据的方式更新缓存。对于这些更新，您可以使用`options.update`函数。

`options.update`需要两个参数。第一个是[`DataProxy`][]对象的一个​​实例，它有一些方法可以让您与商店中的数据进行交互。第二个是您的突变的响应 - 乐观的响应或服务器返回的实际响应。

为了更改您的[`DataProxy`][]实例上的存储调用方法中的数据，如[`writeQuery`] []和[`writeFragment`] []。这将更新您的缓存并反应性地重新呈现任何查询受影响数据的GraphQL组件。

要读取您正在更改的商店中的数据，请确保使用[`DataProxy`][]的诸如[`readQuery`] []和[`readFragment`] []的方法。

有关使用`options.update`函数进行突变之后更新缓存的更多信息，请务必阅读[Apollo Client技术文档相关的主题](../core/read-and-write.html#updating-the-cache-after-a-mutation)。

[`DataProxy`]: ../core/apollo-client-api.html#DataProxy
[`writeQuery`]: ../core/apollo-client-api.html#DataProxy.writeQuery
[`writeFragment`]: ../core/apollo-client-api.html#DataProxy.writeFragment
[`readQuery`]: ../core/apollo-client-api.html#DataProxy.readQuery
[`readFragment`]: ../core/apollo-client-api.html#DataProxy.readFragment

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

有时，当您进行突变时，您还需要更新查询中的数据，以便用户可以看到最新的用户界面。有更多细微的方式来更新缓存中的数据，包括[`options.updateQueries`](#graphql-mutation-options-updateQueries)和[`options.update`](#graphql-mutation-options-update)。但是，您可以通过使用`options.refetchQueries`以更高的效率更高效地更新缓存中的数据。

`options.refetchQueries`将使用您的网络接口执行一个或多个查询，然后将这些查询的结果归一化到缓存中。允许您可能会重新提取您之前提取的查询，或者获取全新的查询。

`options.refetchQueries`是一个字符串或对象的数组。

如果`options.refetchQueries`是一个字符串数组，那么Apollo Client将寻找与提供的字符串相同名称的任何查询，并使用它们当前的变量来获取这些查询。因此，例如，如果您有一个名为`Comments`的查询的GraphQL查询组件（查询可能看起来像：`query Comments {...}`），并且将包含 `Comments` 的字符串数组传递给`options.refetchQueries`，那么`Comments`查询将被重新执行，当它解析最新的数据将被反映在你的UI中。

如果`options.refetchQueries`是一个对象的数组，那么对象必须有两个属性：

- `query`: Query是一个必需的属性，它接受使用`graphql-tag` 的 `gql` 标记创建的GraphQL查询。它应该包含一个单独的GraphQL查询操作，一旦突变完成，它将被执行。
- `[variables]`: 当 `query` 接受一些变量时，这是一个可选的变量对象。

如果指定了具有此形状的对象数组，那么Apollo Client将使用它们的变量来提取这些查询。

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

**注意：我们建议使用[update](#graphql-mutation-options-update) 而不是`updateQueries`。**

此选项允许您根据突变的结果更新您的 Store。默认情况下，Apollo Client将更新您商店中的所有重叠节点。与您定义的 `dataIdFromObject` 返回的共享ID相同的任何内容将使用您的突变结果的新字段进行更新。但是，有时这个还不够。有时您可能希望以取决于缓存中当前数据的方式更新缓存。对于这些更新，您可以使用`options.updateQueries`函数。

`options.updateQueries`需要一个对象，其中查询名称是关键字，而reducer函数是这些值。如果您熟悉Redux，定义 `options.updateQueries` reducers与定义Redux reducer非常相似。对象看起来像这样：

```js
{
  Comments: (previousData, { mutationResult, queryVariables }) => nextData,
}
```

确保您的`options.updateQueries`对象的键对应于您在应用程序中的其他位置的实际查询。查询名称将是在指定`query`操作类型后放置的名称。所以例如在以下查询中：

```graphql
query Comments {
  entry(id: 5) {
    comments {
      ...
    }
  }
}
```

查询名称将为 `Comments`。如果您的应用程序中某个地方尚未执行一个名为 `Comments` 的GraphQL查询，那么reducer函数将永远不会由Apollo运行，`options.updateQueries` 中的键/值对将被忽略。

您提供的函数的第一个参数作为对象的值将是您的查询的先前数据。所以如果你的键是 `Comments`，那么第一个参数将是你的`Comments`查询返回的最后一个数据对象，或者是使用`Comments`查询的任何组件呈现的当前对象。

函数值的第二个参数是具有三个属性的对象：

- `mutationResult`：`mutationResult` 属性将表示您在点击服务器后的突变结果。如果你提供了一个[`options.optimisticResponse`](#graphql-mutation-options-optimisticResponse)，那么`mutationResult`可能就是那个对象。
- `queryVariables`：执行查询的最后一组变量。这是有帮助的，因为当您指定查询名称时，它将只更新当前变量集存储中的数据。
- `queryName`：这是您正在更新的查询的名称。它与您为`options.updateQueries`提供的密钥相同。

你的`options.updateQueries`函数的返回值_必须_与你的第一个`previousData`参数的结构相同。但是，你_绝不能_ 更改 `previousData` 对象。相反，您必须使用更改创建一个新对象。就像Redux reducer。

要了解有关`options.updateQueries`的更多信息，请阅读我们的使用说明文件[使用`updateQueries`控制 Store](cache-updates.html#updateQueries)。

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
        // 注意我们如何返回一个新的 `previousData` 副本，而不是使它变异。这就像一个Redux reducer！
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
