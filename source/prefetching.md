---
title: 预取数据
---

预取是通过 Apollo 客户端能够使应用的 UI 感觉更快的最简单的方法之一。预取仅仅意味着在数据需要在屏幕上呈现之前将其加载到缓存中。本质上，一旦我们猜到用户将导航至某个视图时，我们想要加载其所需的所有数据。

在 Apollo 客户端中，预取非常简单，可以在渲染组件的查询之前运行它。这有一个简单的例子，在 GitHunt 中，一旦用户将鼠标悬停在指向评论页的链接上时，我们就会使用 `withApollo` 高阶组件直接请求一个 `query`。在数据被预取的情况下，评论页将立即呈现，用户通常不会感受到一丁点儿延迟：

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

有很多不同的方法来预测用户会在 UI 交互中的哪一步需要数据。除了使用鼠标悬停状态，这里还有一些其他可以预加载数据的地方：

1. 紧接执行多步骤向导的下一步之后
2. 引导性按钮的路径
3. 在应用的某个子区域内导航时，即刻获取该区域中的所有数据

如果你有一些其他想法，请发送 PR 到这篇文章，也可以添加一些代码片段。预取的一种特殊形式是[服务端 store 注水](/react/server-side-rendering.html#store-rehydration)，因此你可能还会考虑注入比首屏加载实际所需更多的数据，以使其他数据交互更快。
