---
title: 突变
---

除了使用查询获取数据，Apollo还可以帮助您处理GraphQL突变。在GraphQL中，突变与语法中的查询相同，唯一的区别是您使用关键字 `mutation` 而不是 `query` 来表示此查询的根字段将对后端执行写入操作。

```js
mutation {
  submitRepository(repoFullName: "apollographql/apollo-client") {
    id
    repoName
  }
}
```

GraphQL突变在一个查询字符串中代表两件事情：

1.变量字段名称，带有参数`submitRepository`，表示在服务器上要完成的实际操作。
2.您想要从突变结果中更新客户端的字段，在这种情况下为 `{ id, repoName }`。

上述突变将向GitHunt提交一个新的GitHub存储库，将一个条目保存到数据库中。结果可能是：

```
{
  "data": {
    "submitRepository": {
      "id": "123",
      "repoName": "apollographql/apollo-client"
    }
  }
}
```

当我们在Apollo中使用突变时，结果通常会[根据结果的id](cache-updates.html＃dataIdFromObject)自动整合到缓存中，而这又自动更新UI，所以我们经常不需要明确处理结果。为了让客户端正确地执行此操作，我们需要确保在结果中选择必要的字段。一个好的策略可以是简单地询问可能受到突变影响的任何领域。或者，您可以使用[fragments](fragments.html)共享查询和更新该查询的突变之间的字段。

<h2 id="basics">基本突变</h2>

使用 `graphql` 与突变可以很容易地将操作绑定到您的组件。不同于查询，它提供了一个具有大量元数据和方法的复杂对象，突变只能在一个称为 `mutate` 的属性中为包装组件提供一个简单的功能。

```js
import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

class NewEntry extends Component { ... }

const submitRepository = gql`
  mutation submitRepository {
    submitRepository(repoFullName: "apollographql/apollo-client") {
      createdAt
    }
  }
`;

const NewEntryWithData = graphql(submitRepository)(NewEntry);
```

如果我们为上面的组件编写 `propTypes`，它们将如下所示：

```js
NewEntry.propTypes = {
  mutate: PropTypes.func.isRequired,
};
```

<h2 id="calling-mutations">调用突变</h2>

大多数突变将需要查询变量形式的参数，您也可以提供其他选项。 [参见API文档中的完整突变选项集](api-mutations.html＃graphql-mutation-options)

最简单的选择是在包装组件中调用它时直接将选项传递给默认的 `mutate` 属性：

```js
import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

class NewEntry extends Component {
  onClick() {
    this.props.mutate({
      variables: { repoFullName: 'apollographql/apollo-client' }
    })
      .then(({ data }) => {
        console.log('got data', data);
      }).catch((error) => {
        console.log('there was an error sending the query', error);
      });
  }

  render() {
    return <div onClick={this.onClick.bind(this)}>Click me</div>;
  }
}

const submitRepository = gql`
  mutation submitRepository($repoFullName: String!) {
    submitRepository(repoFullName: $repoFullName) {
      createdAt
    }
  }
`;

const NewEntryWithData = graphql(submitRepository)(NewEntry);
```

<h3 id="custom-arguments">自定义参数</h3>

虽然上述方法与默认支持工作正常，通常你会想要保持格式化变化选项从你的演示组件的关注。最好的方法是使用[`props`](api-graphql.html＃graphql-config-props)配置将突变包装在一个完全接受需要的参数的函数中：

```js
const NewEntryWithData = graphql(submitRepository, {
  props: ({ mutate }) => ({
    submit: (repoFullName) => mutate({ variables: { repoFullName } }),
  }),
})(NewEntry);
```

这是在一个组件的上下文中，现在可以简单得多，因为它只需要传递一个参数：

```js
import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

const NewEntry = ({ submit }) => (
  <div onClick={() => submit('apollographql/apollo-client')}>
    Click me
  </div>
);

const submitRepository = gql`...`; // 与上述相同的查询

const NewEntryWithData = graphql(submitRepository, {
  props: ({ mutate }) => ({
    submit: (repoFullName) => mutate({ variables: { repoFullName } }),
  }),
})(NewEntry);
```

请注意，一般来说，您不需要直接使用突变回调的结果。相反，您通常应该依赖于Apollo的基于id的缓存更新来为您处理。如果这不能满足您的需求，[有几个不同的选项可以在更新 store 后发生变化](cache-updates.html)。这样，您可以将UI组件保持为无状态并尽可能声明性。

<h2 id="multiple-mutations">多重突变</h2>

如果组件上需要多个突变，则可以为每个组件创建一个graphql容器：

```js
const ComponentWithMutations =
  graphql(submitNewUser, { name: 'newUserMutation' })(
    graphql(submitRepository, { name: 'newRepositoryMutation' })(Component)
  )
```

确保使用[`graphql()` 容器上的 `name` 选项](api-graphql.html＃graphql-config-name)来命名提供的属性，这样两个容器都不会同时命名他们的`mutate`函数。

如果你想要语法更优雅，考虑使用[`compose`](api-graphql.html＃compose)：

```js
import { compose } from 'react-apollo';

const ComponentWithMutations = compose(
  graphql(submitNewUser, { name: 'newUserMutation' }),
  graphql(submitRepository, { name: 'newRepositoryMutation' })
)(Component);
```

这与前面的代码段完全相同，但是使用更好的语法来平衡事物。

<h2 id="optimistic-ui">乐观UI</h2>

有时您的客户端代码即使在服务器响应结果之前也可以很容易地预测成功突变的结果。例如，在GitHunt中，当用户对存储库进行评论时，我们希望立即在UI中显示新的注释，而不用等待来回服务器的延迟，为用户提供更快的用户界面体验。这就是我们所说的[Optimistic UI](optimistic-ui.html)。如果客户可以预测突变的*乐观响应*，则可以通过Apollo进行。

所有您需要做的是指定 `optimisticResponse` 选项。这个“假结果”将用于立即更新活动查询，就像服务器的突变响应一样。乐观补丁存储在缓存中的一个单独的位置，所以一旦实际的突变返回，相关的乐观更新被自动抛弃，并被替换成真实的结果。

```js
import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

class CommentPage extends Component { ... }
CommentPage.propTypes = {
  submit: PropTypes.func.isRequired,
};

const submitComment = gql`
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

const CommentPageWithData = graphql(submitComment, {
  props: ({ ownProps, mutate }) => ({
    submit: ({ repoFullName, commentContent }) => mutate({
      variables: { repoFullName, commentContent },

      optimisticResponse: {
        __typename: 'Mutation',
        submitComment: {
          __typename: 'Comment',
          // 注意，如果我们需要这些信息来计算乐观的响应，我们可以访问“ownProps”的容器的属性
          postedBy: ownProps.currentUser,
          createdAt: +new Date,
          content: commentContent,
        },
      },
    }),
  }),
})(CommentPage);
```

对于上面的例子，很容易构造乐观的响应，因为我们知道新注释的形状，并且可以大致预测创建的数据。乐观的反应不一定是完全正确的，因为它总是会被服务器的真正的结果所取代，但它应该足够接近，使用户觉得没有任何延迟。

<h2 id="mutation-results">设计突变结果</h2>

当人们谈论GraphQL时，他们经常关注事物的数据获取方面，因为这就是GraphQL带来的最大价值。突变可以非常优雅，如果做得好，但设计好突变的原则，特别是良好的突变结果类型，在开源社区尚未被很好的理解。因此，当您使用突变时，您可能经常觉得您需要做出很多应用程序决定。

在GraphQL中，突变可以返回任何类型，并且可以像常规GraphQL查询一样查询该类型。所以问题是 - 什么样的特定突变返回？

在大多数情况下，从突变结果可获得的数据应该是服务器开发人员对客户端需要了解服务器发生的情况的数据的最佳猜测。例如，在博客文章上创建新评论的突变可能会返回评论本身。重新排列阵列的突变可能需要返回整个数组。

<h2 id="update-after-mutation">存储更新</h2>

大多数情况下，不需要告诉Apollo缓存的哪些部分进行更新，但如果您的突变正在创建新对象或删除某些内容，则需要编写一些额外的逻辑。 请阅读[关于更新store的文章](cache-updates.html)。

有关React Apollo for GraphQL突变支持的所有选项和功能的详细信息，请务必查看[`graphql()` 突变 API 参考](api.html#mutations)。
