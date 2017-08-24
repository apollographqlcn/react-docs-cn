---
title: 分页
---

通常，在你的应用中有些视图，显示的列表需要包含太多数据，而无法一次性获取或显示。分页是解决该问题的最常见的方案，Apollo 客户端通过内置的功能，使其变得非常简单。

获取分页数据基本上有两种方法：编号页面和游标。还有两种显示分页数据的方法：离散页面和无限滚动。如果想更深入地了解它们的不同之处，以及各自的使用场景，建议你阅读我们撰写的关于该主题的博文：[了解分页](https://medium.com/apollo-stack/understanding-pagination-rest-graphql-and-relay-b10f835549e7)。

在本文中，我们将介绍使用 Apollo 实现这两种分页方法的技术细节。

<h2 id="fetch-more">使用 `fetchMore`</h2>

在Apollo中，最简单的分页方法是使用一个叫做 [`fetchMore`](cache-updates.html#fetchMore) 的函数，它由 `graphql` 高阶组件的 `data` prop 提供，它基本上允许你执行一个新的 GraphQL 查询并将结果合并到原始结果中。

你可以指定要用于新查询的查询和变量，以及如何将新查询结果与客户端上的现有数据进行合并。你的设置将决定你会实现哪种分页。

<h2 id="numbered-pages">基于偏移量</h2>

基于偏移量的分页 - 也称为编号页面 - 是一种在许多网站上非常常见的模式，因为通常是后端最容易实现的方式。例如，在 SQL 中，使用 [OFFSET 和 LIMIT](https://www.postgresql.org/docs/8.2/static/queries-limit.html) 可以很容易地生成编号页面。

以下是从 [GitHunt](https://github.com/apollographql/GitHunt-React) 获取的编号页面的示例：

```js

const FEED_QUERY = gql`
  query Feed($type: FeedType!, $offset: Int, $limit: Int) {
    currentUser {
      login
    }
    feed(type: $type, offset: $offset, limit: $limit) {
      id
      # ...
    }
  }
`;

const ITEMS_PER_PAGE = 10;
const FeedWithData = graphql(FEED_QUERY, {
  options(props) {
    return {
      variables: {
        type: (
          props.params &&
          props.params.type &&
          props.params.type.toUpperCase()
        ) || 'TOP',
        offset: 0,
        limit: ITEMS_PER_PAGE,
      },
      fetchPolicy: 'network-only',
    };
  },
  props({ data: { loading, feed, currentUser, fetchMore } }) {
    return {
      loading,
      feed,
      currentUser,
      loadMoreEntries() {
        return fetchMore({
          // 查询： ...（你可以指定一个不同的查询，默认情况下使用 FEED_QUERY）
          variables: {
            // 我们可以指定偏移量的值，因为它与 feed 长度相匹配，但是我们也可以使用状态或者先前的变量来计算（参见下面的游标示例）
            offset: feed.length,
          },
          updateQuery: (previousResult, { fetchMoreResult }) => {
            if (!fetchMoreResult) { return previousResult; }
            return Object.assign({}, previousResult, {
              // 将新的 feed 结果追加到旧的 feed 结果
              feed: [...previousResult.feed, ...fetchMoreResult.feed],
            });
          },
        });
      },
    };
  },
})(Feed);
```

[在 GitHunt 的提交中查看这段代码。](https://github.com/apollographql/GitHunt-React/blob/c0b18795a18b3da42dc90cf7c63b29b14965206d/ui/Feed.js#L165)

你可以看到，`fetchMore` 可以通过 `props` 函数的 `data` 参数访问到。所以我们的演示组件可以不用知道 Apollo，我们使用 `props` 定义一个简单的 “load more” 函数，名为 `loadMoreEntries`，可以由子组件 `Feed` 调用。这样，如果我们需要改变分页逻辑，我们就不需要改变 `Feed` 组件了。

在上面的例子中，`loadMoreEntries` 是一个函数，它调用 `fetchMore`，把当前 feed 的长度作为一个变量。默认情况下，`fetchMore` 更多会使用原来的 `query`，所以我们只是传入新的变量。一旦从服务器返回新数据，`updateQuery` 函数用于将它与现有数据进行合并，这将导致你的 UI 组件使用扩展后的列表重新渲染。

下面的代码展示了如何从 UI 组件调用 `loadMoreEntries` 函数：

```js
const Feed = ({ vote, loading, currentUser, feed, loadMoreEntries }) => {
  return (
    <div>
      <FeedContent
        entries={feed || []}
        currentUser={currentUser}
        onVote={vote}
      />
      <a onClick={loadMoreEntries}>Load more</a>
      {loading ? <Loading /> : null}
    </div>
  );
}
```

上述方法适用于限量/偏移分页。使用编号页面或偏移量的分页的一个缺点是，当某一项同时插入列表或从列表中移除时，该项可能会被跳过或返回两次。这可以通过基于游标的分页来避免。

<h2 id="cursor-pages">基于游标</h2>

在基于游标的分页中，使用“游标”来跟踪数据集中应该从哪获取下一组项的位置。有时候，游标可能非常简单，只是引用所获取的最后一个对象的ID，但在某些情况下，例如根据某些条件排序的列表，游标需要对排序条件进行编码以作为最后一个对象的ID的附加信息。

在客户端上实现基于游标的分页与基于偏移的分页没有什么不同，我们保留对所获取的最后一个对象的引用以及所使用的排序顺序的信息，而不是使用绝对偏移量。

在下面的示例中，我们使用 `fetchMore` 查询来连续加载新的评论，这些评论将被添加到列表前。在 `fetchMore` 查询中使用的游标由最初服务器的响应中提供，并且每当获取更多数据时都会更新。

```js
const MoreCommentsQuery = gql`
  query MoreComments($cursor: String) {
    moreComments(cursor: $cursor) {
      cursor
      comments {
        author
        text
      }
    }
  }
`;

const CommentsWithData = graphql(Comment, {
  // 这个函数在每次 `data` 变化时都会重新执行，包括 `updateQuery` 之后，这意味着我们的 loadMoreEntries 函数将始终具有正确的游标
  props({ data: { loading, cursor, comments, fetchMore } }) {
    return {
      loading,
      comments,
      loadMoreEntries: () => {
        return fetchMore({
          query: MoreCommentsQuery,
          variables: {
            cursor: cursor,
          },
          updateQuery: (previousResult, { fetchMoreResult }) => {
            const previousEntry = previousResult.entry;
            const newComments = fetchMoreResult.moreComments.comments;

            return {
              // 通过这里返回的 `cursor'，我们将为 `loadMore` 函数更新新的游标。
              cursor: fetchMoreResult.cursor,

              entry: {
                // 将新的评论放在列表的前面
                comments: [...newComments, ...previousEntry.entry.comments],
              },
            };
          },
        });
      },
    };
  },
})(Feed);
```

<h2 id="relay-cursors">Relay 式的游标分页</h2>

Relay 作为另一个流行的 GraphQL 客户端，对分页查询的输入和输出有专门的处理方式，所以人们有时会根据 Relay 的要求构建服务器的分页模型。如果你有一个服务器被设计为使用 [Relay 游标连接](https://facebook.github.io/relay/graphql/connections.htm) 规范，你也可以从 Apollo 客户端正常调用该服务器。

使用 Relay 式的游标与基于游标的基本分页非常类似。主要区别在于影响游标位置的查询响应的格式。

Relay 在返回的游标连接上提供了一个 `pageInfo` 对象，该对象包含的游标，分别把返回的第一个和最后一个项作为属性 `startCursor` 和 `endCursor`。此对象还包含一个布尔属性 `hasNextPage`，可用于确定是否有更多可用结果。

以下示例一次指定10个项的请求，并且返回的结果应开始于提供的 `cursor` 之后。如果游标的值为 `null` 则 relay 将忽略它，并从数据集开始处提供结果，该数据集允许对初始请求和后续请求使用相同的查询。

```js
const CommentsQuery = gql`
  query Comments($cursor: String) {
    Comments(first: 10, after: $cursor) {
      comments {
        edges {
          node {
            author
            text
          }
        }
        pageInfo {
          endCursor
          hasNextPage
        }
      }
    }
  }
`;

const CommentsWithData = graphql(CommentsQuery, {
  // 这个函数在每次 `data` 变化时重新运行，包括`updateQuery`之后，这意味着我们的 loadMoreEntries 函数将始终有正确的游标
  props({ data: { loading, comments, fetchMore } }) {
    return {
      loading,
      comments,
      loadMoreEntries: () => {
        return fetchMore({
          query: CommentsQuery,
          variables: {
            cursor: comments.pageInfo.endCursor,
          },
          updateQuery: (previousResult, { fetchMoreResult }) => {
            const newEdges = fetchMoreResult.comments.edges;
            const pageInfo = fetchMoreResult.comments.pageInfo;

            return {
              // 将新的评论放在列表末尾并更新 `pageInfo`，这样我们就可以得到新的 `endCursor` 和 `hasNextPage` 的值
              comments: {
                edges: [...previousResult.comments.edges, ...newEdges],
                pageInfo,
              },
            };
          },
        });
      },
    };
  },
})(Feed);
```

<h2 id="connection-directive">`@connection` 指令</h2>
当使用分页查询时，累积查询的结果可能难以在 store 中查找，因为传递给查询的参数用于标识默认的 store 键，但通常在执行查询的代码之外无法访问到。这对于命令式的 store 更新是有问题的，因为没有用于更新目标的可靠的 store 键。要引导 Apollo 客户端为分页查询使用可靠的 store 键，你可以使用可选的 `@ connection` 指令为部分查询指定 store 键。例如，如果我们想要提前提供一个可靠的 store 键，我们可以调整我们的查询以使用 `@connection` 指令：
```
const FEED_QUERY = gql`
  query Feed($type: FeedType!, $offset: Int, $limit: Int) {
    currentUser {
      login
    }
    feed(type: $type, offset: $offset, limit: $limit) @connection(key: "feed", filter: ["type"]) {
      id
      # ...
    }
  }
`;
```

这将导致每个查询或 `fetchMore` 中累积的 feed 被放置在 store 的 `feed` 键下，之后我们便可以使用命令式 store 更新。在本例中，我们还使用了 `@connection` 指令的可选 `filter` 参数，它允许我们在 store 键中加入查询的一些参数。上面的例子中，我们希望在 store 键中包含 `type` 查询参数，这样可以产生多个 store 值，它们从每种类型的 feed 页面的数据累积而来。