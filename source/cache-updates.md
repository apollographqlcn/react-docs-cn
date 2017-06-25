---
title: 更新 Store
---

Apollo执行两个重要的核心任务：执行查询和突变，并缓存结果。

感谢Apollo的商店设计，查询或变更的结果可能会在所有正确的地方更新您的UI。在许多情况下，这可能会自动发生，而在其他情况下，您需要帮助客户端这样做。

<h2 id="normalization">使用 `dataIdFromObject` 进行范式化</h2>

阿波罗根据两件事情做出缓存：

1. GraphQL查询的形状及其结果。
2. 从服务器返回的对象的标识。

基于对象标识将缓存平坦化称为高速缓存归一化。您可以在我们的博客文章["GraphQL Concepts Visualized"](https://medium.com/apollo-stack/the-concepts-of-graphql-bc68bd819be3)中详细阅读我们的缓存模型。

默认情况下，Apollo基于两个属性来标识对象：`__typename`和一个ID字段，可以是`id`或`_id`。客户端会自动将`__typename`字段添加到查询中，因此您必须确保获取`id`字段（如果有）。

```js
// 这个结果
{
  __typename: 'Person',
  id: '1234',
  name: 'Jonas',
}

// 将获得以下ID
'Person:1234'
```

您还可以指定一个自定义函数，从每个对象生成ID，并将其作为[`ApolloClient`构造函数](initialization.html＃creation-client)中的`dataIdFromObject`提供，如果要指定阿波罗将如何识别和取消复制从服务器返回的对象。

```js
import { ApolloClient } from 'react-apollo';

// 如果您的数据库具有所有类型对象的唯一ID，您可以使用非常简单的功能！
const client = new ApolloClient({
  dataIdFromObject: o => o.id
});
```

这些ID允许阿波罗客户端反应地告知所有查询特定对象关于该部分商店更新的查询。

如果要获取dataIdFromObjectFunction（例如使用[`readFragment`函数](/core/apollo-client-api.html＃ApolloClient\.readFragment)），可以以`client.dataIdFromObject` 方式访问它。

```js
const person = {
  __typename: 'Person',
  id: '1234',
};

client.dataIdFromObject(person); // 'Person:1234'
```

<h3 id="automatic-updates">自动存储更新</h3>

我们来看一下刚刚使用缓存归一化的情况，导致我们商店的正确更新。假设我们做以下查询：

```graphql
{
  post(id: '5') {
    id
    score
  }
}
```

然后，我们进行以下突变：

```graphql
mutation {
  upvotePost(id: '5') {
    id
    score
  }
}
```

如果两个结果的 `id` 字段匹配，那么我们的UI中的 `score` 字段将自动更新！尽可能利用此属性的一个好方法是使您的突变结果具有更新先前提取的查询所需的所有数据。一个简单的技巧是使用[fragments](fragments.html)来共享查询和影响它的突变之间的字段。

<h2 id="after-mutations">突变后更新</h2>

在某些情况下，只需使用 `dataIdFromObject` 即可更新应用程序UI。例如，如果要在不重写整个列表的情况下将对象添加到对象列表中，或者如果存在无法分配对象标识符的某些对象，则Apollo Client无法为您更新现有查询。继续阅读，了解您可以使用的其他工具。

<h3 id="refetchQueries">`refetchQueries`</h3>

`refetchQueries`是更新缓存的最简单的方法。使用`refetchQueries`，您可以指定一个或多个要在突变完成后运行的查询，以便重新获取可能受突变影响的存储部分：

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

使用`refetchQueries`的一个很常见的方法是导入为其他组件定义的查询，以确保这些组件将被更新：

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

使用`update`可以完全控制缓存，从而可以根据您喜欢的任何方式更改数据模型。 `update`是在查询后更新缓存的推荐方法。完整地解释了[这里](http://dev.apollodata.com/react/api-mutations.html#graphql-mutation-options-update)。

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
            // 将我们的评论从突变添加到最后。
            data.comments.push(submitComment);
            // 将我们的数据写回缓存。
            store.writeQuery({ query: CommentAppQuery, data });
          },
        });
      },
    };
  },
})(CommentsPage);
```

<h3 id="updateQueries">`updateQueries`</h3>

**注意：我们建议使用更灵活的 `update` API而不是 `updateQueries`。 `updateQueries` API将来可能会被弃用。**

顾名思义，`updateQueries`可以根据突变的结果来更新你的UI。要重新强调：大多数情况下，只要对象ID与您的商店中已存在的ID相匹配，您的UI会根据突变结果自动更新。有关如何利用此功能的更多信息，请参阅上述[`normalization`](#normalization)文档。

但是，如果要删除或添加项目到具有突变的列表或不能将对象标识符分配给相关对象，则必须使用 `updateQueries` 确保您的UI正确反映了更改。

我们将以GitHunt中的评论页为例。当我们提交一个新的评论时，“提交”按钮将触发一个突变，在服务器上保留的注释的“列表”中添加一个新的注释。实际上，服务器不知道有一个列表 - 它只是知道在SQL中的 `comments` 表中添加了一些东西，所以服务器不能真正地告诉我们确切的结果。获取列表注释的原始查询也不了解此新评论，因此Apollo无法自动将其添加到列表中。

在这种情况下，我们可以使用 `updateQueries` 来确保查询结果更新，这也将更新阿波罗的规范化存储，使所有内容保持一致。

如果您熟悉Redux，请将 `updateQueries` 选项作为reducer，除了直接更新商店之外，我们正在更新查询结果形状，这意味着我们不必担心商店内部工作如何工作。

我们通过 `CommentsPage` 组件可以调用的函数prop来暴露这个突变。这是代码如下所示：

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

如果我们仔细查看服务器架构，我们会看到这个突变实际上返回了有关添加的单个新注释的信息;它不会提取整个注释列表。这很有意义：如果我们在页面上有一千条评论，如果我们添加一个新的评论，我们不想重新获取所有评论。

评论页面本身使用以下查询呈现：

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

现在，我们必须将由突变返回的新添加的注释合并到已经在页面加载时被触发的`COMMENT_QUERY`返回的信息中。我们通过`updateQueries`完成这个。放大该部分代码：

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

从根本上说，`updateQueries` 是一个从查询名称（在我们的例子中是 `Comment`）的地图，它接收到这个查询接收到的先前结果以及突变返回结果的函数。在我们的例子中，突变返回有关新注释的信息。该函数应该将突变结果合并到包含查询先前接收的结果（`prev`）的新对象中，并返回该新对象。

请注意，该函数不能更改`prev`对象（因为`prev`与返回的新对象进行比较，以查看功能所做的更改，因此需要更新）。

在`Comment`查询的`updateQueries`函数中，我们正在做一些非常简单的事情：只需将我们刚刚提交的注释添加到查询所要求的注释列表中。我们正在使用[immutability-helper](https://github.com/kolodny/immutability-helper)软件包中的`update`函数，只是简单的做。但是，如果你想，你可以编写一些不需要帮助的Javascript来将两个传入的对象组合成一个新的对象。

一旦突变触发并且结果从服务器到达（或者通过乐观UI提供结果），则将调用`Comment`查询的`updateQueries`函数，并且相应地更新`Comment`查询。结果中的这些更改将映射到React道具，我们的UI将会更新新的信息！

<h3 id="resultReducers">查询 `reducer`</h3>

**注意：我们建议使用更灵活的`update`API而不是`reducer`。 `reducer`API将来可能会被废弃。**

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
