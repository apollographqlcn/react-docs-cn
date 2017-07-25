---
title: 更新 Store
---

Apollo 的核心任务中重要的有两个：执行查询和突变，并缓存结果。

得益于 Apollo 的 store 设计，查询或突变的结果能在所有必要的地方更新您的 UI。在大多数情况下，这个过程会被自动执行，而在某些情况下，您需要手动指定客户端执行更新。

<h2 id="normalization">使用 `dataIdFromObject` 范式化缓存</h2>

Apollo 基于如下两点执行缓存：

1. GraphQL 查询文档的结构及其响应。
2. 从服务端返回的对象的标识。

基于对象标识将缓存扁平化称为缓存范式化。您可以在我们的博客文章 ["GraphQL 概念可视化"](https://medium.com/apollo-stack/the-concepts-of-graphql-bc68bd819be3) 中详细阅读我们的缓存模型。

默认情况下，Apollo 基于两个属性来标识对象：`__typename` 和一个 ID 字段，可以是 `id` 或 `_id`。客户端会自动将 `__typename` 字段添加到查询中，因此您必须确保获取 `id` 字段（如果有的话）。

```js
// 该结果...
{
  __typename: 'Person',
  id: '1234',
  name: 'Jonas',
}

// ...将获得以下 ID
'Person:1234'
```

如果要指定 Apollo 对服务端返回的对象如何标识和去重，您还可以指定一个自定义函数，从各个对象中生成 ID，并将其作为 [`ApolloClient` 构造函数](initialization.html＃creation-client)中的 `dataIdFromObject` 值。

```js
import { ApolloClient } from 'react-apollo';

// 如果您的数据库具有所有类型对象的唯一 ID，您可以如下提供一个简易的函数
const client = new ApolloClient({
  dataIdFromObject: o => o.id
});
```

这些 ID 允许 Apollo 客户端指定所有查询获取特定对象以更新该部分 store。

如果要获取 dataIdFromObjectFunction（例如使用 [`readFragment` 函数](/core/apollo-client-api.html＃ApolloClient\.readFragment)），可以以`client.dataIdFromObject` 方式访问它。

```js
const person = {
  __typename: 'Person',
  id: '1234',
};

client.dataIdFromObject(person); // 'Person:1234'
```

<h3 id="automatic-updates">自动更新 store</h3>

我们来看一下刚刚使用范式化缓存使 store 得以正确更新的情况。假设我们执行以下查询：

```graphql
{
  post(id: '5') {
    id
    score
  }
}
```

然后，我们执行以下突变：

```graphql
mutation {
  upvotePost(id: '5') {
    id
    score
  }
}
```

如果两个结果的 `id` 字段匹配，那么我们的 UI 中的 `score` 字段将自动更新！尽可能利用此特性能使您的突变结果更新之前查询获取的相应数据。其中一个简单的技巧是使用 [片段](fragments.html) 来共享查询和突变之间的字段。

<h2 id="after-mutations">突变后的更新</h2>

在某些情况下，只需使用 `dataIdFromObject` 即可更新应用 UI。例如，如果要在不重写整个列表的情况下将对象添加到对象列表中，或者如果存在无法指定对象标识符的某些对象，则 Apollo 客户端无法为您更新现有查询。请继续阅读下文以了解您可以使用的其他方法。

<h3 id="refetchQueries">`refetchQueries`</h3>

`refetchQueries` 是更新缓存的最简单方法。使用 `refetchQueries`，您可以指定一个或多个要在突变完成后执行的查询，以便重新获取可能受突变影响的部分 store 的数据：

```javascript
mutate({
  // ...插入注释突变
  refetchQueries: [{
    query: gql`
      query updateCache($repoName: String!) {
        entry(repoFullName: $repoName) {
          id
          comments {
            postedBy {
              login
              html_url
            }
            createdAt
            content
          }
        }
      }
    `,
    variables: { repoFullName: 'apollographql/apollo-client' },
  }],
})
```

`refetchQueries` 的一个常见用法是导入为其他组件定义的查询，以确保这些组件将被更新：

```javascript
import RepoCommentsQuery from '../queries/RepoCommentsQuery';

mutate({
  // ...插入注释突变
  refetchQueries: [{
    query: RepoCommentsQuery,
    variables: { repoFullName: 'apollographql/apollo-client' },
  }],
})
```


<h3 id="directAccess">`update`</h3>

使用 `update` 可以完全控制缓存，从而可以根据您喜欢的任意方式更改数据模型。如果要在查询后更新缓存，那么推荐 `update` 方法。[这里](http://dev.apollodata.com/react/api-mutations.html#graphql-mutation-options-update)能找到完整示意。

```javascript
import CommentAppQuery from '../queries/CommentAppQuery';

const SUBMIT_COMMENT_MUTATION = gql`
  mutation submitComment($repoFullName: String!, $commentContent: String!) {
    submitComment(repoFullName: $repoFullName, commentContent: $commentContent) {
      postedBy {
        login
        html_url
      }
      createdAt
      content
    }
  }
`;

const CommentsPageWithMutations = graphql(SUBMIT_COMMENT_MUTATION, {
  props({ ownProps, mutate }) {
    return {
      submit({ repoFullName, commentContent }) {
        return mutate({
          variables: { repoFullName, commentContent },

          update: (store, { data: { submitComment } }) => {
            // 从我们的缓存中读取此查询的数据。
            const data = store.readQuery({ query: CommentAppQuery });
            // 将新增的评论从突变中添加到缓存数据列表末尾。
            data.comments.push(submitComment);
            // 将更改后的数据写回缓存。
            store.writeQuery({ query: CommentAppQuery, data });
          },
        });
      },
    };
  },
})(CommentsPage);
```

<h3 id="updateQueries">`updateQueries`</h3>

**注意：我们建议使用更灵活的 `update` API而不是 `updateQueries`。 `updateQueries` API将来可能会被废弃。**

顾名思义，`updateQueries` 可以根据突变的结果来更新你的 UI。再次重申：大多数情况下，只要对象ID与您的商店中已存在的ID相匹配，您的 UI 会根据突变结果自动更新。更多有关如何利用此功能的信息，请参阅上述[`范式化`](#normalization)文档。

但是，如果使用突变删除或添加项目至列表，或无法为相关对象指定对象标识符，则必须使用 `updateQueries` 确保您的 UI 正确反映了更新。

我们将以 GitHunt 中的评论页为例。当我们提交一个新的评论时，“提交”按钮将触发一个突变，往服务端的评论“列表”中添加一条新的评论。实际上，服务端并不知道有这样一个列表 - 它只是知道向 SQL 中的 `comments` 表中添加一些数据，所以服务端无法告诉我们确切的结果。最初获取注释列表的查询也无法预知该条新评论，因此 Apollo 无法自动将其添加到列表中。

在这种情况下，我们可以使用 `updateQueries` 来确保更新查询结果，同时更新 Apollo 的范式化缓存，使所有内容保持一致。

如果您熟悉Redux，请将 `updateQueries` 选项作为 reducer，除了直接更新 store 之外，还能更新查询结果的结构，这意味着我们不必关心 store 内部是如何工作的。

我们通过 `CommentsPage` 组件可以调用的函数 prop 来封装这个突变。代码如下所示：

```javascript
import update from 'immutability-helper';

const SUBMIT_COMMENT_MUTATION = gql`
  mutation submitComment($repoFullName: String!, $commentContent: String!) {
    submitComment(repoFullName: $repoFullName, commentContent: $commentContent) {
      postedBy {
        login
        html_url
      }
      createdAt
      content
    }
  }
`;

const CommentsPageWithMutations = graphql(SUBMIT_COMMENT_MUTATION, {
  props({ ownProps, mutate }) {
    return {
      submit({ repoFullName, commentContent }) {
        return mutate({
          variables: { repoFullName, commentContent },

          updateQueries: {
            Comment: (prev, { mutationResult }) => {
              const newComment = mutationResult.data.submitComment;

              return update(prev, {
                entry: {
                  comments: {
                    $unshift: [newComment],
                  },
                },
              });
            },
          },
        });
      },
    };
  },
})(CommentsPage);
```

如果我们仔细查看服务端的 schema，我们会看到这个突变实际上返回的是即将被添加的单个注释的信息；它不会重新获取整个注释列表。这么做是有道理的：假设我们在页面上有一千条评论，如果要添加一条新的评论，那么不必重新获取所有评论。

评论页由以下查询结果渲染而成：

```javascript
const COMMENT_QUERY = gql`
  query Comment($repoName: String!) {
    currentUser {
      login
      html_url
    }

    entry(repoFullName: $repoName) {
      id
      postedBy {
        login
        html_url
      }
      createdAt
      comments {
        postedBy {
          login
          html_url
        }
        createdAt
        content
      }
      repository {
        full_name
        html_url
        description
        open_issues_count
        stargazers_count
      }
    }
  }`;
```

现在，我们必须将突变返回的新添加的评论，合并到页面加载时已被触发的 `COMMENT_QUERY` 返回的数据中。我们通过 `updateQueries` 实现该功能。代码大致如下：

```javascript
mutate({
  //...
  updateQueries: {
    Comment: (prev, { mutationResult }) => {
      const newComment = mutationResult.data.submitComment;
      return update(prev, {
        entry: {
          comments: {
            $unshift: [newComment],
          },
        },
      });
    },
  },
})
```

本质上，`updateQueries` 是一个从查询的名称（本例中是 `Comment`），到接收这个查询及突变返回结果作为参数的函数的映射关系。在本例中，突变返回有关新增评论的信息。然后该函数将突变结果合并到之前查询返回的结果（`prev`）对象中，并返回该新对象。

请注意，该函数中不能更改 `prev` 对象（因为要用到 `prev` 与返回的新对象进行比较，以检查该函数更新点，来决定哪些 props 需要更新）。

在 `Comment` 查询的 `updateQueries` 函数中，我们要实现的功能非常简单：只需将我们刚刚提交的评论添加到查询所返回的评论列表中。我们只是简单地使用了 [immutability-helper](https://github.com/kolodny/immutability-helper) 模块中的 `update` 函数。当然，如果你愿意的话，你可以编写一些不需要帮助函数的 JS 代码来将两个传入的对象合并成一个新的对象。

一旦突变触发并且从服务端获取结果之后（或者通过乐观 UI 提供结果），则将调用 `Comment` 查询的 `updateQueries` 函数，并且相应地更新 `Comment` 查询的结果。结果的变更将映射到 React 的 props，我们的 UI 也将会更新新的信息！

<h3 id="resultReducers">查询 `reducer`</h3>

**注意：我们建议使用更灵活的 `update` API 而不是 `reducer`。 `reducer` API 将来可能会被废弃。**

`updateQueries` 和 `update` 只能根据突变的结果来更新其他查询，`reducer`选项可以让您根据任何Apollo操作更新查询结果，包括其他查询的结果，突变或订阅。它的行为就像一个Reduce reducer对非规范化的查询结果：

```javascript
import update from 'immutability-helper';

const CommentsPageWithData = graphql(CommentsPageQuery, {
  props({ data }) {
    // ...
  },
  options({ params }) {
    return {
      reducer: (previousResult, action, variables) => {
        if (action.type === 'APOLLO_MUTATION_RESULT' && action.operationName === 'submitComment'){
          // 注意：通常建议采用一些更健全的检查方法，以确保previousResult不为空，并且突变结果包含我们预期的数据。

          // 注意：变量包含当前的查询变量，而不是导致操作的查询或变量的变量。那些可以在`action`参数中找到。

          return update(previousResult, {
            entry: {
              comments: {
                $unshift: [action.result.data.submitComment],
              },
            },
          });
        }
        return previousResult;
      },
    };
  },
})(CommentsPage);
```

您可以看到，`reducer` 选项可用于实现与`updateQueries`相同的目标，但它更灵活，适用于任何类型的阿波罗动作，而不仅仅是突变。例如，可以基于另一查询的结果来更新查询结果。

>目前无法响应您的自定义Redux操作，从Abollo外部到达结果缩减器。有关详细信息，请参阅[这个](https://github.com/apollographql/apollo-client/issues/1013)。

**什么时候应该使用update vs. reducer vs. updateQueries vs. refetchQueries？**

应该使用`refetchQueries`，只要突变结果单独不足以推断出缓存的所有更改。 `refetchQueries`也是一个非常好的选择，如果一个额外的往返和可能的过时提取不是您的应用程序所关心的，这在原型设计中通常是正确的。与`update`，`updateQueries`和`reducer`相比，`refetchQueries`是最容易编写和维护的。

`updateQueries`，`reducer`和`update`都提供类似的功能。他们按照这个顺序进行介绍，每个人都试图解决前一个问题的缺点。虽然目前可以使用所有三个API，但我们强烈建议尽可能使用`update`，因为将来可能会弃用其他两个API（`updateQueries`和`reducer`）。我们建议`更新`，因为它的API是最强大和最容易理解的三个。我们考虑弃用`reducer`和`updateQueries`的原因是它们都依赖于客户端的内部状态，这使得它们比`update`更难理解和维护，而不需要提供任何额外的功能。

<h2 id="fetchMore">增量加载： `fetchMore`</h2>

`fetchMore`可以根据另一个查询返回的数据来更新查询的结果。大多数情况下，它用于处理无限卷动分页或其他情况，当您已经有一些时，您正在加载更多的数据。

在我们的GitHunt示例中，我们有一个分页的feed显示GitHub存储库的列表。当我们点击“加载更多”按钮时，我们不希望Apollo Client丢弃已经加载的存储库信息。相反，它应该将新加载的存储库附加到Apollo Client已经在商店中的列表中。通过此更新，我们的UI组件应该重新渲染并显示所有可用的存储库。

让我们看一下如何使用`fetchMore`方法对查询进行处理：

```javascript
const FeedQuery = gql`
  query Feed($type: FeedType!, $offset: Int, $limit: Int) {
    # ...
  }`;

const FeedWithData = graphql(FeedQuery, {
  props({ data: { loading, feed, currentUser, fetchMore } }) {
    return {
      loading,
      feed,
      currentUser,
      loadNextPage() {
        return fetchMore({
          variables: {
            offset: feed.length,
          },

          updateQuery: (previousResult, { fetchMoreResult }) => {
            if (!fetchMoreResult) { return previousResult; }

            return Object.assign({}, previousResult, {
              feed: [...previousResult.feed, ...fetchMoreResult.feed],
            });
          },
        });
      },
    };
  },
})(Feed);
```

这里有两个组件：`FeedWithData`和`Feed`。 `FeedWithData` 容器实现产生了将`props`传递给演示文稿`Feed`组件。具体来说，我们将`loadNextPage`的参数映射到以下内容：

```js
return fetchMore({
  variables: {
    offset: feed.length,
  },
  updateQuery: (prev, { fetchMoreResult }) => {
    if (!fetchMoreResult.data) { return prev; }
    return Object.assign({}, prev, {
      feed: [...prev.feed, ...fetchMoreResult.feed],
    });
  },
});
```

`fetchMore`方法使用新查询发送的`变量`映射。在这里，我们将偏移量设置为`feed.length`，以便我们获取未在Feed中显示的项目。该变量映射与为与组件相关联的查询指定的映射合并。这意味着其他变量，例如。 `limit`变量的值与组件查询中的值相同。

它也可以使用一个`query`命名的参数，它可以是一个GraphQL文档，其中包含一个将被提取以获取更多信息的查询;我们将其称为`fetchMore`查询。默认情况下，`fetchMore`查询是与容器关联的查询，在这种情况下`FEED_QUERY`。

当我们调用`fetchMore`时，Apollo Client将触发`fetchMore`查询，并使用`updateQuery`选项中的逻辑把它合并到原来的结果中。命名参数`updateQuery`应该是一个函数，它接收与您的组件相关联的查询的先前结果（即在这种情况下为`FEED_QUERY`）以及`fetchMore`查询返回的信息，并返回两者的组合。

这里，`fetchMore`查询与与该组件关联的查询相同。我们的`updateQuery`会返回新的Feed项，并将它们附加到我们以前要求的Feed项目中。这样，UI将会更新，Feed将包含下一页的项目！

虽然`fetchMore`通常用于分页，但还有许多其他适用的情况。例如，假设您有一个项目列表（例如，协作待办事项列表），并且您有一种方式来获取在一定时间后更新的项目。然后，您不必重新获取整个待办事项列表即可获取更新：只要新增的项目与`fetchMore`相结合，只要您的`updateQuery`函数正确地合并新结果即可。

<h3 id="connection-directive">The `@connection` directive</h3>
By default, the result of a `fetchMore` will be stored in the cache according to the initial query executed and its parameters. Due to this behavior, it can be hard to know the location in the cache to run an imperative update on if the variables from the initial query are not known, which often happens if we are running store updates from a different place than where the queries are executed.

To have a stable cache location for query results, Apollo Client 1.6 introduced the `@connection` directive, which can be used to specify a custom store key for results. To use the `@connection` directive, simply add the directive to the segment of the query you want a custom store key for and provide the `key` parameter to specify the store key. In addition to the `key` parameter, you can also include the optional `filter` parameter, which takes an array of query argument names to include in the generated custom store key.

```
const query = gql`query Feed($type: FeedType!, $offset: Int, $limit: Int) {
  feed(type: $type, offset: $offset, limit: $limit) @connection(key: "feed", filter: ["type"]) {
    ...FeedEntry
  }
}`
```

With the above query, even with multiple `fetchMore`s, the results of each feed update will always result in the `feed` key in the store being updated with the latest accumulated values. In this example, we also use the `@connection` directive's optional `filter` argument to include the `type` query argument in the store key, which results in multiple store values that accumulate queries from each type of feed.

Now that we have a stable store key, we can easily use `writeQuery` to perform a store update, in this case clearing out the feed.

```
client.writeQuery({
  query: gql`
    query Feed($type: FeedType!) {
      feed(type: $type) @connection(key: "feed", filter: ["type"]) {
        id
      }
    }
  `,
  variables: {
    type: "top",
  },
  data: {
    feed: [],
  },
});
```

Note that because we are only using the `type` argument in the store key, we don't have to provide `offset` or `limit`.

<h2 id="cacheRedirect">使用`customResolvers`缓存重定向</h2>

在某些情况下，查询会以不同的密钥请求客户端存储中已存在的数据。一个非常常见的例子是当您的UI具有列表视图和使用相同数据的详细视图。列表视图可能会运行以下查询：

```
query ListView {
  books {
    id
    title
    abstract
  }
}
```

当选择特定书籍时，详细信息视图将显示使用此查询的单个项目：

```
query DetailView {
  book(id: $id) {
    id
    title
    abstract
  }
}
```

我们知道数据很可能已经在客户端缓存中，但是由于使用不同的查询请求，Apollo Client不知道。为了告诉Apollo Client在哪里查找数据，我们可以定义自定义解析器：

```
import ApolloClient, { toIdValue } from 'apollo-client';

const client = new ApolloClient({
  networkInterface,
  customResolvers: {
    Query: {
      book: (_, args) => toIdValue(dataIdFromObject({ __typename: 'book', id: args['id'] })),
    },
  },
  dataIdFromObject,
});
```

Apollo Client将使用自定义解析器的返回值来查找其缓存中的项目。必须使用`toIdValue`来表示返回的值应该被解释为一个id，而不是一个标量值或一个对象。此示例中的`Query`键是您的根查询类型名称。

还可以返回一个ID列表：

```
customResolvers: {
  Query: {
    books: (_, args) => args['ids'].map(id =>
      toIdValue(dataIdFromObject({ __typename: 'book', id: id }))),
  },
},
```
