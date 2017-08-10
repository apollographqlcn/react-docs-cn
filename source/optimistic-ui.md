---
title: Optimistic UI
---

如[mutations](mutations.html#optimistic-ui)部分所述，optimistic UI是一种模式，你可以使用它来模拟突变的结果，甚至在从服务器接收响应之前更新UI。一旦从服务器接收到响应，乐观的结果将被丢弃并替换为实际结果。

乐观的UI提供了一种简单的方法，使你的UI响应更快，同时确保数据在实际响应达到时保持一致。

<h2 id =“optimistic-basics”>基本 optimistic UI </h2>

假设我们有一个“编辑评论”突变，我们希望UI在用户提交突变时立即更新，而不是等待服务器响应。这是mutate函数提供的 `optimisticResponse` 参数。

使用Apollo获取GraphQL数据到UI组件的主要方法是使用查询，因此如果我们希望我们的乐观响应更新UI，我们必须确保返回一个乐观的响应，以更新正确的查询结果。使用[`dataIdFromObject`](cache-updates.html#dataIdFromObject) 选项进一步了解如何执行此操作。

以下是代码中的内容：

```js

const updateComment = gql`
  mutation updateComment($commentId: ID!, $commentContent: String!) {
    updateComment(commentId: $commentId, commentContent: $commentContent) {
      id
      __typename
      content
    }
  }
`;

const CommentPageWithData = graphql(submitComment, {
  props: ({ ownProps, mutate }) => ({
    submit({ commentId, commentContent }) {
      return mutate({
        variables: { commentId, commentContent },
        optimisticResponse: {
          __typename: 'Mutation',
          updateComment: {
            id: commentId,
            __typename: 'Comment',
            content: commentContent,
          },
        },
      });
    },
  }),
})(CommentPage);
```

我们选择 `id` 和 `__typename`，因为这是我们的 `dataIdFromObject` 用来确定一个全局唯一的对象ID。我们需要确保为这些字段提供正确的值，以便Appollo知道我们所指的对象。

<h2 id="optimistic-advanced">添加到列表</h2>

在上面的例子中，我们展示了如何使用乐观的突变结果无缝地编辑 Store 中的现有对象。然而，许多突变不只是更新存储中的现有对象，而是插入一个新对象。

在这种情况下，我们需要指定如何将新数据集成到现有查询中，从而将UI整合到我们的UI中。你可以在有关[控制 Store](cache-updates.html) 的文章中详细阅读有关如何做的事情 - 特别是我们可以使用`update`函数将结果插入到现有查询的结果集中。 `update`对于乐观的结果和从服务器返回的真实结果的工作方式完全相同，所以就像上面我们只需要添加 `optimisticResponse` 选项。

这是 GitHunt 的一个具体例子，它将注释插入现有的注释列表。

```js
import React from 'react';
import { gql, graphql } from 'react-apollo';
import update from 'immutability-helper';

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
          optimisticResponse: {
            __typename: 'Mutation',
            submitComment: {
              __typename: 'Comment',
              postedBy: ownProps.currentUser,
              createdAt: +new Date,
              content: commentContent,
            },
          },
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
