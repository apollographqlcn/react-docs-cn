---
title: 突变
---

除了使用查询获取数据，Apollo 还可以帮助你处理 GraphQL 突变。在 GraphQL 中，突变与它语法中的查询相同，唯一的区别是你使用关键字 `mutation` 而不是 `query` ，表明此查询的根字段将对后端执行写入操作。

```js
mutation {
  submitRepository(repoFullName: "apollographql/apollo-client") {
    id
    repoName
  }
}
```

GraphQL 突变在一个查询字符串中代表两件事情：

1. 带参数的突变字段名称，`submitRepository`，表示在服务端要完成的实际操作。
2. 你想要从突变结果中更新客户端的字段，在这种情况下为 `{ id, repoName }`。

上述突变将向 GitHunt 提交一个新的 GitHub 仓库，保存一个条目到数据库中。返回的结果可能是：

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

当我们在Apollo中使用突变时，返回的响应结果通常会[根据该结果的 id](cache-updates.html＃dataIdFromObject) 自动整合到缓存中，而这又会自动更新 UI，所以我们经常不需要明确处理结果。为了让客户端正确地执行此操作，我们需要确保在结果中选择必要的字段。一个好的策略可以是简单地请求可能受到突变影响的任何字段。或者，你可以使用[片段](fragments.html)共享查询和更新该查询的突变之间的字段。

<h2 id="basics">基本突变</h2>

使用 `graphql` 与突变可以很容易地将操作绑定到你的组件。不同于查询，它提供了一个具有大量元数据和方法的复杂对象，突变只能在一个称为 `mutate` 的属性中为包裹组件提供一个简单的功能。

```js
import React, { Component } from 'react';
import PropTypes from 'prop-types';
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

大多数突变都需要的参数形如查询使用的变量，你也可以提供其他选项。 [参见API文档中的完整突变选项集](api-mutations.html＃graphql-mutation-options)

最简单的选择是在包裹组件中调用它时直接将选项传递给默认的 `mutate` 属性：

```js
import React, { Component } from 'react';
import PropTypes from 'prop-types';
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

虽然上述方法采用的是默认 props，但通常情况下，你会想要格式化展示组件中的突变选项，实现关注点分离。最好的方法是使用 [`props`](api-graphql.html＃graphql-config-props) 配置将突变包装在一个只接受需要的参数的函数中：

```js
const NewEntryWithData = graphql(submitRepository, {
  props: ({ mutate }) => ({
    submit: (repoFullName) => mutate({ variables: { repoFullName } }),
  }),
})(NewEntry);
```

这是一个带上下文的一个组件，现在简单明了，因为它只需要传递一个参数：

```js
import React, { Component } from 'react';
import PropTypes from 'prop-types';
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

请注意，一般来说，你不需要直接使用突变回调的结果。相反，通常情况下，你应该依赖于 Apollo 的基于 id 的缓存来为你处理更新。如果这不能满足你的需求，[有几个不同的选项可以在突变之后更新 store](cache-updates.html)。这样，你可以将 UI 组件保持为无状态并尽可能声明性。

<h2 id="multiple-mutations">多个突变</h2>

如果组件上需要多个突变，则可以为每个组件创建一个 graphql 容器：

```js
const ComponentWithMutations =
  graphql(submitNewUser, { name: 'newUserMutation' })(
    graphql(submitRepository, { name: 'newRepositoryMutation' })(Component)
  )
```

确保使用 [`graphql()` 容器上的 `name` 选项](api-graphql.html＃graphql-config-name)来命名提供的属性，这样两个容器都不会同时命名它们的 `mutate` 函数。

如果你想要语法更优雅，考虑使用[`compose`](api-graphql.html＃compose)：

```js
import { compose } from 'react-apollo';

const ComponentWithMutations = compose(
  graphql(submitNewUser, { name: 'newUserMutation' }),
  graphql(submitRepository, { name: 'newRepositoryMutation' })
)(Component);
```

这段代码与之前的作用完全相同，但使用了更优雅的语法来平衡功能与可读性。

<h2 id="optimistic-ui">Optimistic UI</h2>

有时你的客户端代码即使在服务器响应结果返回之前也可以很容易地预测成功突变的结果。例如，在 GitHunt 中，当用户对存储库进行评论时，我们希望立即在UI中显示新的评论，而不用等待服务器的延迟，为用户提供更快的用户界面体验。这就是我们所说的 [Optimistic UI](optimistic-ui.html)。如果客户端可以预测突变的 *Optimistic 响应*，则可以通过 Apollo 实现 Optimistic UI。

你需要做的就只是指定 `optimisticResponse` 选项。这个“假结果”将用于立即更新当前的查询，就像服务器的突变响应一样。Optimistic 补丁存储在缓存中一个单独的位置，所以一旦实际的突变结果返回，相关的 Optimistic 更新将被自动删除，并被替换成真实的响应结果。

```js
import React, { Component } from 'react';
import PropTypes from 'prop-types';
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
          // 注意，如果我们需要这些信息来计算 Optimistic 响应，我们可以访问 `ownProps` 的容器属性
          postedBy: ownProps.currentUser,
          createdAt: +new Date,
          content: commentContent,
        },
      },
    }),
  }),
})(CommentPage);
```

对于上面的例子，很容易构造 Optimistic 响应，因为我们知道新注释的结构，并且可以大致预测创建的数据。Optimistic 响应不一定是完全正确的，因为它总是会被服务器的真正的结果所取代，但它应该足够接近真实的响应，使用户感觉不到任何延迟。

<h2 id="mutation-results">设计突变结果</h2>

当人们谈论 GraphQL 时，他们经常关注代码的数据获取有关的知识，因为这就是GraphQL带来的最大价值。如果设计得当，突变可以非常优雅，但设计好突变的原则，特别是好的突变结果类型，在开源社区尚未达成共识。因此，当你使用突变时，你可能经常觉得你需要根据应用场景做出决定。

在 GraphQL 中，突变可以返回任何类型，并且可以像常规 GraphQL 查询一样查询该类型。所以问题是 - 一个特定的突变应该返回什么样的类型？

在大多数情况下，从突变结果返回的数据应该是服务端开发人员对客户端需要了解的服务端操作的数据的最佳猜测。例如，在博客文章上创建新评论的突变可能会返回评论本身，重新排列数组的突变可能需要返回整个数组。

<h2 id="update-after-mutation">存储更新</h2>

大多数情况下，没必要指定 Apollo 更新哪部分缓存，但如果你的突变正在创建新对象或删除某些内容，则需要编写一些额外的逻辑。 请阅读[关于更新store的文章](cache-updates.html)。

有关 React Apollo for GraphQL 突变支持的所有选项和功能的详细信息，请务必查看[`graphql()` 突变 API 参考](api.html#mutations)。
