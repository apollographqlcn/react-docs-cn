---
title: 使用片段
---

[GraphQL片段](http://graphql.org/learn/queries/#fragments) 是一个共享的查询逻辑片段。

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

Apollo 片段有两种主要用途：
  - 在多个查询，突变或订阅之间共享字段。
  - 打破您的查询，让您可以将现场访问与其使用的位置进行共同定位。

在本文中，我们将概述两者的模式;我们还将使用[`graphql-anywhere`](https://github.com/apollographql/graphql-anywhere)和[`graphql-tag`](https://github.com/apollographql/graphql-tag)中的实用程序旨在帮助我们，特别是第二个问题。

<h2 id="reusing-fragments">复用片段</h2>

片段的最直接的使用是在应用程序的各个部分重用部分查询（或突变或订阅）。例如，在评论页面的 GitHunt 中，我们希望在发布评论之后获取相同的字段，因为我们最初的查询。这样，我们可以确保在数据更改时呈现一致的注释对象。

为此，我们可以简单地分享一个描述我们需要注释的字段的片段：

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

我们把这个片段放在 `​​CommentsPage.fragments.comment` 上，使用熟悉的 `gql` 助手来创建它。

当将时间片段嵌入到查询中时，我们只需在GraphQL中使用 `...Name` 语法，并将该片段嵌入到查询GraphQL文档中：

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

您可以在GitHunt [这里](https://github.com/apollographql/GitHunt-React/blob/master/ui/routes/CommentsPage.js)查看 `CommentsPage` 的完整源代码。

<h2 id="colocating-fragments">协调片段</h2>

GraphQL的一个关键优点是响应数据的树状特征，在许多情况下，这些属性反映了您渲染的组件层次结构。这与GraphQL对片段的支持相结合，可以让您将查询分开，使得查询获取的各个字段位于使用该字段的代码旁边。

虽然这种技术并不总是有意义的（例如，GraphQL架构并不总是由UI要求驱动），但是如果这样做，可以使用Apollo客户端中的一些模式来充分利用它。

在GitHunt中，我们在[`FeedPage`](https://github.com/apollographql/GitHunt-React/blob/master/ui/routes/FeedPage.js) 上显示了一个例子，它构建了后续视图层次结构：

```
FeedPage
└── Feed
    └── FeedEntry
        ├── RepoInfo
        └── VoteButtons
```

`FeedPage` 进行查询以获取 `Entry` 列表，每个子组件需要每个 `Entry` 的不同子字段。

`graphql-anywhere` 包为我们提供了一个工具来轻松构建一个单一查询，该查询提供了每个子组件需要的所有字段，并允许轻松地传递组件需要的确切字段。

<h3 id="creating-fragments">创建片段</h3>

要创建碎片，我们再次使用`gql`帮助器并附加到 `ComponentClass.fragment` 的子字段，例如：

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

`graphql-anywhere` 包给我们提供的一个不错的工具是一个[`PropType`](https://facebook.github.io/react/docs/reusable-components.html) 检查器，我们可以使用它来确保我们确实在组件的 `entry` 道具中接收这些字段：

```js
import { propType } from 'graphql-anywhere';

VoteButtons.propTypes = {
  // ...
  entry: propType(VoteButtons.fragments.entry).isRequired,
};
```

如果我们的碎片包含子片段，那么我们可以将它们传递给 `gql` 助手：

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

我们还可以使用 `graphql-anywhere` 包从 `entry` 中过滤出确切的字段，然后再将它们传递给子组件。所以当我们渲染一个 `VoteButtons` 时，我们可以简单地做：

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

<h3 id="webpack-importing-fragments" title="Fragments with Webpack">使用 Webpack 时导入片段</h3>

当使用[graphql-tag/loader](https://github.com/apollographql/graphql-tag/blob/master/loader.js)加载 `.graphql` 文件时，我们可以使用 `import` 语句包括片段。例如：

When loading `.graphql` files with [graphql-tag/loader](https://github.com/apollographql/graphql-tag/blob/master/loader.js), we can include fragments using `import` statements. For example:

```graphql
#import "./someFragment.graphql"
```

将使 `someFragment.graphql` 的内容可用于当前文件。有关其他详细信息，请参阅[Webpack Fragments](webpack.html#Fragments)部分。
