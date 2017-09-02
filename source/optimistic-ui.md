---
title: 乐观 UI
---

As explained in the mutations section, optimistic UI is a pattern that you can use to simulate the results of a mutation and update the UI even before receiving a response from the server. Once the response is received from the server, optimistic result is thrown away and replaced with the actual result.

Optimistic UI provides an easy way to make your UI respond much faster, while ensuring that the data becomes consistent with the actual response when it arrives.

如 [mutations](mutations.html#optimistic-ui) 部分所述，乐观 UI 是一种你可以使用它来模拟突变结果模式，你甚至可以在服务器接收响应之前更新 UI。一旦从服务器接收到响应，乐观结果将被丢弃并替换为实际结果。

乐观 UI 提供了一种使你的 UI 响应更快的简单方法，同时确保数据在实际响应返回时保持一致。

<h2 id =“optimistic-basics”>基本的乐观 UI</h2>

假设我们有一个“编辑评论”的突变，我们希望 UI 在用户提交突变时立即更新，而不是等待服务器响应。这便是 mutate 函数提供的 `optimisticResponse` 参数。

使用 Apollo 获取 GraphQL 数据到 UI 组件的主要方法是使用查询，因此如果我们希望使用乐观响应更新 UI，我们必须确保返回一个乐观响应，以更新正确的查询结果。进一步了解如何使用 [`dataIdFromObject`](cache-updates.html#dataIdFromObject) 选项执行此操作。

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

const CommentPageWithData = graphql(updateComment, {
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

我们选择 `id` 和 `__typename`，因为这是我们的 `dataIdFromObject` 用来确定一个全局唯一的对象 ID。我们需要确保为这些字段提供正确的值，以便Appollo 知道我们所指的对象。

<h2 id="optimistic-advanced">添加到列表</h2>

在上面的例子中，我们展示了如何使用乐观突变结果无缝地编辑 store 中的现有对象。然而，许多突变不只是更新存储中的现有对象，而是插入一个新对象。

在这种情况下，我们需要指定如何将新数据集成到现有查询中，从而更新我们的 UI。你可以详细阅读有关 [操控 store](cache-updates.html) 的文章中是如何做的 - 特别是我们可以使用 `update` 函数将结果插入到现有查询的结果集中。 `update` 对于乐观结果和从服务器返回的真实结果的处理方式完全相同。

这是 GitHunt 的一个具体例子，它将一条评论插入现有的评论列表。

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
            // 将我们突变的评论添加到最后。
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
