---
title: 预取数据
---

预取是使 Apollo Client 能够使应用程序的UI感觉更快的最简单的方法之一。预取仅仅意味着在数据需要在屏幕上呈现之前将数据加载到缓存中。基本上，我们想要加载视图所需的所有数据，一旦我们猜到用户将导航到它。

在 Apollo Client 中，预取非常简单，可以在渲染组件的查询之前运行它。作为一个简单的例子，在 GitHunt 中，一旦用户将鼠标悬停在指向注释页面的链接上，我们就会使用 `withApollo` 高阶组件直接调用 `query`。在预取数据的情况下，注释页面立即呈现，用户通常不会有延迟：

```js
const FeedEntry = ({ entry, currentUser, onVote, client }) => {
  const repoLink = `/${entry.repository.full_name}`;
  const prefetchComments = (repoFullName) => () => {
    client.query({
      query: COMMENT_QUERY,
      variables: { repoName: repoFullName },
    });
  };

  return (
    <div className="media">
      ...
      <div className="media-body">
        <RepoInfo
          description={entry.repository.description}
          stargazers_count={entry.repository.stargazers_count}
          open_issues_count={entry.repository.open_issues_count}
          created_at={entry.createdAt}
          user_url={entry.postedBy.html_url}
          username={entry.postedBy.login}
        >
          <Link to={repoLink} onMouseOver={prefetchComments(entry.repository.full_name)}>
              View comments ({entry.commentCount})
          </Link>
        </RepoInfo>
      </div>
    </div>
  );
};

const FeedEntryWithApollo = withApollo(FeedEntry);
```

有很多不同的方法来预测用户将最终在UI中需要一些数据。除了使用悬停状态，这里还有一些其他可以预加载数据的地方：

1. 立即执行多步骤向导的下一步
2. 号召性用语按钮的路线
3. 应用程序的一个子区域的所有数据，即可在该区域内导航

如果你有其他一些想法，请发送PR到这篇文章，也许添加一些更多的代码片段。预取的一种特殊形式是[服务端 Store 水化](/react/server-side-rendering.html#store-rehydration)，因此你可能还会考虑比首页加载实际需要更多的数据，以使其他数据交互更快。
