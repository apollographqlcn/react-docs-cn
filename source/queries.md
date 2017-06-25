---
title: 查询
---

在此页面上，您可以学习如何使用 `react-apollo` 将GraphQL查询结果附加到您的React UI。本指南介绍了对GraphQL本身的熟悉程度。您可以在[graphql.org](http://graphql.org/docs/queries/)上详细了解GraphQL查询。

我们的核心价值观之一就是“它只是GraphQL”。当使用 `react-apollo` 时，您不需要学习关于查询语法的任何特别的东西，因为所有内容都只是标准的GraphQL。您可以在GraphiQL查询IDE中输入任何内容，也可以将其添加到 `react-apollo` 代码中。

<h2 id="basics">基本查询</h2>

当我们运行一个基本的查询时，我们可以用一个非常简单的方式使用 `graphql` 容器。我们只需使用 `gql` 模板文字解析我们的查询，然后将其作为第一个参数传递给`graphql`容器。

例如，在GitHunt中，我们要在 `Profile` 组件中显示当前登录的用户：

```js
import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

class Profile extends Component { ... }

// 我们使用gql标签来将查询字符串解析成查询文档
const CurrentUserForLayout = gql`
  query CurrentUserForLayout {
    currentUser {
      login
      avatar_url
    }
  }
`;

const ProfileWithData = graphql(CurrentUserForLayout)(Profile);
```

当我们用GraphQL查询文档使用 `graphql` 时，会发生两件事情：

1.查询是从Apollo客户端数据存储库中加载的，如果数据不在存储器中，则从服务器加载
2.我们的组件订阅商店，以便如果数据因服务器的突变或某些其他响应而发生变化，则会更新该商店

除了在查询中选择的 `currentUser` 字段之外，`data` prop还包含一个名为 `loading` 的字段，一个布尔值表示当前是否正在从服务器加载该查询。所以如果我们要声明 `propTypes`，他们会看起来像这样：

```js
Profile.propTypes = {
  data: PropTypes.shape({
    loading: PropTypes.bool.isRequired,
    currentUser: PropTypes.object,
  }).isRequired,
};
```

`data.currentUser` 属性将随着客户端了解当前用户随时间的变化而改变。该信息存储在Apollo Client的全局缓存中，因此如果某些其他查询获取有关当前用户的新信息，则此组件将更新以保持一致。您可以在[关于缓存更新的文章](cache-updates.html)中阅读有关使缓存与服务器最新的技术相关的更多信息。

<h2 id="default-result-props" title="The data prop">`data` 属性的结构</h2>

如上所述，`graphql` 将把查询的结果传递给一个名为 `data` 的prop中的包装组件。它也将通过父容器的所有道具。

对于查询，`data` 属性的形状如下所示：

- `...fields`: 查询中每个根字段的一个键。
- `loading`: 如果目前在飞行中有查询提取，包括调用`refetch`后，该字段为 `true`。 `false` 否则。
- `error`: 一个ApolloError对象，表示运行查询时可能发生的不同可能的错误。

还有更多的方法，您可以阅读关于[在API文档中查询](api-queries.html＃graphql-query-data)。例如，对于像这样的查询：

```graphql
query getUserAndLikes($id: ID!) {
  user(userId: $id) { name }
  likes(userId: $id) { count }
}
```

你可以得到如下属性：

```js
data: {
  user: { name: "James" },
  likes: { count: 10 },
  loading: false,
  error: null,
  variables: { id: 'asdf' },
  refetch() { ... },
  fetchMore() { ... },
  startPolling() { ... },
  stopPolling() { ... },
  // ... 等更多方法
}
```

如果你使用 `props` 选项来指定你的子组件的[自定义 `props`](＃graphql-props)，那么这个对象将被传递给 `data` 参数的 `props` 设置。

If you use the `props` option to the wrapper to specify [custom `props`](#graphql-props) for your child component, this object will be passed to the `props` option on the parameter named `data`.

<h2 id="graphql-options">变量和选项</h2>

如果要配置查询，可以在 `graphql` 的第二个参数上提供一个 `options` 键，您的选项将被传递给[`ApolloClient.watchQuery`](/core/apollo-client-api.html＃watchQuery)。如果您的查询需要变量，这是传递他们的地方：

```js
// 假设我们的个人资料查询采用了头像大小
const CurrentUserForLayout = gql`
  query CurrentUserForLayout($avatarSize: Int!) {
    currentUser {
      login
      avatar_url(avatarSize: $avatarSize)
    }
  }
`;

const ProfileWithData = graphql(CurrentUserForLayout, {
  options: { variables: { avatarSize: 100 } },
})(Profile);

```

<h3 id="options-from-props">从属性计算</h3>

通常，查询的变量将从包装器组件的 `props` 中计算出来。无论在应用程序中使用组件的何处，调用者都会传递参数。所以 `options` 可以是将道具传递给组件的一个功能：

```js
// 调用者可以执行以下操作：
<ProfileWithData avatarSize={300} />

// 我们的HOC可能看起来像：
const ProfileWithData = graphql(CurrentUserForLayout, {
  options: ({ avatarSize }) => ({ variables: { avatarSize } }),
})(Profile);
```

默认情况下，`graphql` 将尝试从 `ownProps` 的查询中提取任何缺失的变量。所以在上面的例子中，我们可以使用更简单的 `graphql(CurrentUserForLayout)(Profile);`。但是，如果您需要更改变量的名称，计算该值，或者只是希望对事物进行更明确的定义，那么 `options` 函数就是这样做的地方。

<h3 id="other-graphql-options">其他选项</h3>

除了 `variables` 之外，还有很多其他可以传递的选项，例如 `pollInterval`：

```js
const ProfileWithData = graphql(CurrentUserForLayout, {
  // 查看watchQuery API以获取您可以在这里提供的选项
  options: { pollInterval: 20000 },
})(Profile);
```

如果您使用功能来计算属性中的选项，则每当道具更改时，所有这些 `options` 将自动重新计算。

[阅读API文档中的所有查询选项。](api-queries.html#graphql-query-options)

<h2 id="graphql-skip">跳过操作</h2>

`graphql` 容器API是有意完全静态的，所以你不能在运行时动态地改变查询或包装组件，而不生成一个新的React组件。但是，有时候，您可能需要执行一些条件逻辑来根据传入的道具来跳过查询。为此，您可以使用 `skip` 配置。

例如，如果您想忽略用户未通过身份验证的查询，则可以使用此选项：

```js
const ProfileWithData = graphql(CurrentUserForLayout, {
  skip: (ownProps) => !ownProps.authenticated,
})(Profile);
```

`skip` 也可以是静态属性：

```js
const ProfileWithData = graphql(CurrentUserForLayout, {
  skip: true,
})(Profile);
```

传递 `skip` 配置完全绕过高阶组件，就好像它根本没有。这意味着你的子组件根本没有得到一个 `data` 属性，而 `options` 或 `props` 方法没有被调用。

<h2 id="graphql-props">控制子属性</h2>

默认情况下，与查询一起使用的 `graphql` 将为包装组件提供一个 `data` 属性，其中包含有关查询状态的各种信息。我们还会看到[突变](mutations.html)在`mutate` 属性上提供回调。可以使用这些默认的名称来编写整个应用程序。

但是，如果要将您的UI组件与Apollo分离，并使其在不同的上下文中可重用，您可能需要修改这些默认道具，并将其包含在自己的自定义对象和函数中。

<h3 id="graphql-name">更改属性名称</h3>

如果要更改默认的 `data` 属性的名称，但保持完全相同的形状，可以使用 `graphql` 容器的 `name` 选项。当一个组件通过嵌套的`graphql`容器使用多个查询时，这是特别有用的，其中 `data` 属性将被覆盖。

```js
import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

class Profile extends Component { ... }

Profile.propTypes = {
  CurrentUserForLayout: PropTypes.shape({
    loading: PropTypes.bool.isRequired,
    currentUser: PropTypes.object,
  }).isRequired,
};

const CurrentUserForLayout = gql`
  query CurrentUserForLayout {
    currentUser {
      login
      avatar_url
    }
  }
`;

// 我们希望prop被称为'CurrentUserForLayout'而不是数据
const ProfileWithData = graphql(CurrentUserForLayout, {
  name: 'CurrentUserForLayout'
})(Profile);
```

<h3 id="graphql-props-option">任意转换</h3>

如果要完全控制子组件的道具，请使用 `props` 选项将查询 `data` 对象映射到任何数量的传递给子组件的属性：

```js

import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

// 这里，Profile有一个更通用的API，它没有耦合到Apollo或者我们使用的查询的形状
class Profile extends Component { ... }
Profile.propTypes = {
  userLoading: PropTypes.bool.isRequired,
  user: PropTypes.object,
  refetchUser: PropTypes.func,
};


const CurrentUserForLayout = gql`
  query CurrentUserForLayout {
    currentUser {
      login
      avatar_url
    }
  }
`;

const ProfileWithData = graphql(CurrentUserForLayout, {
  // ownProps是由父组件使用时传递到“ProfileWithData”的属性
  props: ({ ownProps, data: { loading, currentUser, refetch } }) => ({
    userLoading: loading,
    user: currentUser,
    refetchUser: refetch,
  }),
})(Profile);
```

这种使用风格导致您的演示组件(`Profile`)和Apollo之间的最佳解耦。

<h2 id="full-api">完整API</h2>

有关React Apollo for GraphQL查询支持的所有选项和功能的详细信息，请务必查看[关于 `graphql()` 查询的 API 参考](api-queries.html)。
