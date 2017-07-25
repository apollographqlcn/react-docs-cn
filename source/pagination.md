---
title: 分页
---

通常，您将在应用程序中看到一些视图，您需要显示包含太多数据的列表，以便一次性提取或显示。分页是这个问题的最常见的解决方案，Apollo Client具有内置的功能，使其变得非常简单。

基本上有两种获取分页数据的方法：编号页面和光标。还有两种显示分页数据的方法：离散页面和无限滚动。要更深入地解释不同之处，以及何时可以使用一个对象，我们建议您阅读我们关于该主题的博文：[了解分页](https://medium.com/apollo-stack/understanding-pagination-rest-graphql-and-relay-b10f835549e7)。

在本文中，我们将介绍使用Apollo实现这两种方法的技术细节。

<h2 id="fetch-more">使用 `fetchMore`</h2>

在Apollo中，最简单的分页方法是使用一个叫做[`fetchMore`](cache-updates.html#fetchMore)的函数，它由`graphql`高阶组件的`data` prop提供，这基本上允许您执行一个新的GraphQL查询并将结果合并到原始结果中。

您可以指定要用于新查询的查询和变量，以及如何将新查询结果与客户端上的现有数据进行合并。你如何确定将决定你正在实施什么样的分页。

<h2 id="numbered-pages">基于偏移量</h2>

基于偏移量的分页 - 也称为编号页面 - 是许多网站上发现的一种非常常见的模式，因为通常在后台实现最简单。例如，在SQL中，使用[OFFSET和LIMIT](https://www.postgresql.org/docs/8.2/static/queries-limit.html) 可以很容易地生成编号页面。

以下是从[GitHunt](https://github.com/apollographql/GitHunt-React) 获取的编号页面的示例：

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
          // 查询： ...（您可以指定一个不同的查询，默认情况下使用FEED_QUERY）
          variables: {
            // 我们可以弄清楚要使用哪个偏移量，因为它与进给长度相匹配，但是我们也可以使用状态或者先前的变量来计算（参见下面的光标示例）
            offset: feed.length,
          },
          updateQuery: (previousResult, { fetchMoreResult }) => {
            if (!fetchMoreResult) { return previousResult; }
            return Object.assign({}, previousResult, {
              // 将新的Feed结果附加到旧的Feed结果
              feed: [...previousResult.feed, ...fetchMoreResult.feed],
            });
          },
        });
      },
    };
  },
})(Feed);
```

[在GitHunt的上下文中看到这段代码。](https://github.com/apollographql/GitHunt-React/blob/c0b18795a18b3da42dc90cf7c63b29b14965206d/ui/Feed.js#L165)

你可以看到，`fetchMore` 可以通过`props`函数的`data`参数访问到。所以我们的演示组件可以不知道Apollo，我们使用`props`定义一个简单的 “load more” 函数，名为`loadMoreEntries`，可以由子组件`Feed`调用。这样，如果我们需要改变分页逻辑，我们就不需要改变 `Feed` 组件了。

在上面的例子中，`loadMoreEntries`是一个函数，它调用`fetchMore`，把当前feed的长度作为一个变量。默认情况下，`fetchMore`更多的会使用原来的`query`，所以我们只是传入新的变量。一旦从服务器返回新数据，`updateQuery`函数用于将它与现有数据进行合并，这将导致您的UI组件重新渲染扩展列表。

以下是从UI组件调用`loadMoreEntries`函数的方式：

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

上述方法适用于极限/偏移分页。具有编号页面或偏移量的分页的一个缺点是，当项目同时插入列表或从列表中移除时，项目可以跳过或返回两次。这可以通过基于光标的分页来避免。

<h2 id="cursor-pages">基于游标</h2>

在基于游标的分页中，使用“游标”来跟踪数据集中应该从中获取下一个项目的位置。有时候，游标可能非常简单，只是引用所取得的最后一个对象的ID，但在某些情况下，例如根据某些条件排序的列表，游标需要对排序条件进行编码以及最后一个对象的ID牵强。

在客户端上实现基于游标的分页与基于偏移的分页没有什么不同，而不是使用绝对偏移，我们保留对所提取的最后一个对象的引用以及所使用的排序顺序的信息。

在下面的示例中，我们使用`fetchMore`查询来连续加载新的注释，这些注释将被添加到列表中。在 `fetchMore` 查询中使用的光标在初始服务器响应中提供，并且每当获取更多数据时都会更新。

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
  // 这个函数在每次`data`变化时重新运行，包括`updateQuery`之后，这意味着我们的 loadMoreEntries 函数将始终有正确的游标
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
              // 在这里返回`cursor'，我们将`loadMore`函数更新为新的游标。
              cursor: fetchMoreResult.cursor,

              entry: {
                // 将新的注释放在列表的前面
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

Relay 作为另一个流行的GraphQL客户端，对分页查询的输入和输出有所了解，所以人们有时会根据 Relay 的需要构建服务器的分页模型。如果您有一个服务器设计为使用[Relay 游标连接](https://facebook.github.io/relay/graphql/connections.htm) 规范，您也可以从Apollo Client 正常调用该服务器。

使用Relay 式的游标非常类似于基于游标的基本分页。主要区别在于影响游标位置的查询响应的格式。

Relay 在返回的光标连接上提供一个 `pageInfo` 对象，该对象包含分别作为属性 `startCursor` 和 `endCursor` 返回的第一个和最后一个项的游标。此对象还包含一个布尔属性 `hasNextPage`，可用于确定是否有更多可用结果。

以下示例一次指定10个项目的请求，并且结果应在提供的 `cursor` 之后开始。如果 `null` 被传递给游标继电器将忽略它，并从数据集开始提供结果，这允许对初始和后续请求使用相同的查询。

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
  // 这个函数在每次`data`变化时重新运行，包括`updateQuery`之后，这意味着我们的loadMoreEntries函数将始终有正确的光标
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
              // 将新的注释放在列表末尾并更新`pageInfo`，这样我们就可以得到新的`endCursor`和`hasNextPage`值
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

<h2 id="connection-directive">The `@connection` directive</h2>
When using paginated queries, results from accumulated queries can be hard to find in the store, as the parameters passed to the query are used to determine the default store key but are usually not known outside the piece of code that executes the query. This is problematic for imperative store updates, as there is no stable store key for updates to target. To direct Apollo Client to use a stable store key for paginated queries, you can use the optional `@connection` directive to specify a store key for parts of your queries. For example, if we wanted to have a stable store key for the feed query earlier, we could adjust our query to use the `@connection` directive:
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

This would result in the accumulated feed in every query or `fetchMore` being placed in the store under the `feed` key, which we could later use of imperative store updates. In this example, we also use the `@connection` directive's optional `filter` argument, which allows us to include some arguments of the query in the store key. In this case, we want to include the `type` query argument in the store key, which results in multiple store values that accumulate pages from each type of feed.