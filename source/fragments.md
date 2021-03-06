---
title: 使用片段
---

[GraphQL 片段](http://graphql.org/learn/queries/#fragments) 是一个代码间共享的查询逻辑片段。

```graphql
fragment NameParts on Person {
  firstName
  lastName
}

query getPerson {
  people(id: "7") {
    ...NameParts
    avatar(size: LARGE)
  }
}
```

Apollo 的片段主要有两种用途：
- 在多个查询，突变或订阅之间共享字段。
- 切分单个查询，让你可以将组件代码和其所需的字段相邻放置。

在本文中，我们将分别概述这两者的模式；我们还将使用 [`graphql-anywhere`](https://github.com/apollographql/graphql-anywhere) 和 [`graphql-tag`](https://github.com/apollographql/graphql-tag) 中的公共方法来提供帮助，特别是在第二种情况下。

<h2 id="reusing-fragments">复用片段</h2>

片段的最直接的使用场景是在应用的各个部分重用部分查询（或突变，或订阅）。例如，在 GitHunt 项目中，对于评论页，我们希望在发布评论之后，能够获取与最初的查询相同的字段。这样，我们可以确保在数据更改时渲染一致的评论对象。

为此，我们可以简单地共享一个描述我们所需评论字段的片段：

```js
import { gql } from 'react-apollo';

CommentsPage.fragments = {
  comment: gql`
    fragment CommentsPageComment on Comment {
      id
      postedBy {
        login
        html_url
      }
      createdAt
      content
    }
  `,
};
```

按照惯例，我们把这个片段放在 `​​CommentsPage.fragments.comment` 里，使用熟悉的 `gql` 辅助函数来创建它。

当需要将片段嵌入到查询中时，我们只需在 GraphQL 中使用 `...Name` 语法，并将该片段嵌入到查询 GraphQL 文本中：

```
const SUBMIT_COMMENT_MUTATION = gql`
  mutation submitComment($repoFullName: String!, $commentContent: String!) {
    submitComment(repoFullName: $repoFullName, commentContent: $commentContent) {
      ...CommentsPageComment
    }
  }
  ${CommentsPage.fragments.comment}
`;

export const COMMENT_QUERY = gql`
  query Comment($repoName: String!) {
    # ...
    entry(repoFullName: $repoName) {
      # ...
      comments {
        ...CommentsPageComment
      }
      # ...
    }
  }
  ${CommentsPage.fragments.comment}
`;
```

你可以在[这里](https://github.com/apollographql/GitHunt-React/blob/master/ui/routes/CommentsPage.js)查看 GitHunt 中 `CommentsPage` 的完整源代码。

<h2 id="colocating-fragments">协调片段</h2>

GraphQL 还有一个主要优点是响应数据的树状特征，在许多情况下，这些属性反映了你渲染的组件的层次结构。这与 GraphQL 对片段的支持相结合，可以让你将查询切分开，使得查询获取的各个字段位于使用该字段的代码旁边。

虽然这种技术并不总是有意义的（例如，GraphQL 的 schema 并不总是由 UI 的需求而驱动设计），但是如果这样做，便可以使用 Apollo 客户端中的一些模式来充分利用它。

在 GitHunt 中，我们在 [`FeedPage`](https://github.com/apollographql/GitHunt-React/blob/master/ui/routes/FeedPage.js) 上展示了一个例子，它构建了如下的视图结构：

```
FeedPage
└── Feed
    └── FeedEntry
        ├── RepoInfo
        └── VoteButtons
```

`FeedPage` 执行查询以获取 `Entry` 列表，每个子组件所需的 `Entry` 又有不同的子字段。

`graphql-anywhere` 包为我们提供了一个工具来轻松构建一个单一查询，该查询提供了每个子组件需要的所有字段，并允许轻易地为组件传递所需的确切字段。

<h3 id="creating-fragments">创建片段</h3>

要创建片段，我们需要再次使用 `gql` 辅助函数并将其赋值给 `ComponentClass.fragment` 的子字段，例如：

```js
VoteButtons.fragments = {
  entry: gql`
    fragment VoteButtons on Entry {
      score
      vote {
        vote_value
      }
    }
  `,
};
```

`graphql-anywhere` 包还给我们提供了一个很棒的工具，它是一个 [`PropType`](https://facebook.github.io/react/docs/reusable-components.html) 检查器，我们可以使用它来确保我们确实能在组件的 `entry` prop 中接收到这些字段：

```js
import { propType } from 'graphql-anywhere';

VoteButtons.propTypes = {
  // ...
  entry: propType(VoteButtons.fragments.entry).isRequired,
};
```

如果我们的片段包含子片段，我们同样可以将它们传递给 `gql` 辅助函数：

```js
FeedEntry.fragments = {
  entry: gql`
    fragment FeedEntry on Entry {
      commentCount
      repository {
        full_name
        html_url
        owner {
          avatar_url
        }
      }
      ...VoteButtons
      ...RepoInfo
    }
    ${VoteButtons.fragments.entry}
    ${RepoInfo.fragments.entry}
  `,
};
```

<h3 id="filtering-with-fragments">使用片段进行过滤</h3>

我们还可以使用 `graphql-anywhere` 包从 `entry` 中过滤出确切的字段，然后再将它们传递给子组件。所以当我们渲染一个 `VoteButtons` 时，我们可以简单地写：

```jsx
import { filter } from 'graphql-anywhere';

<VoteButtons
  entry={filter(VoteButtons.fragments.entry, entry)}
  canVote={loggedIn}
  onVote={type => onVote({
    repoFullName: full_name,
    type,
  })}
/>
```

`filter()` 函数将从片段定义的 `entry` 中获取完整的字段。

<h3 id="webpack-importing-fragments" title="Fragments with Webpack">使用 Webpack 导入片段</h3>

当使用 [graphql-tag/loader](https://github.com/apollographql/graphql-tag/blob/master/loader.js) 加载 `.graphql` 文件时，我们可以使用 `import` 语句引入片段。例如：

```graphql
#import "./someFragment.graphql"
```

这将使 `someFragment.graphql` 的内容应用于当前文件。有关更多详细信息，请参阅 [Webpack Fragments](webpack.html#Fragments) 部分。
