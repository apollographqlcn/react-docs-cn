---
title: 查询
---

本文中，您将学习如何使用 `react-apollo` 把GraphQL查询结果附加到您的 React UI 上。本指南假设您对 GraphQL 本身有一定的了解，您可以在 [graphql.org](http://graphql.org/docs/queries/) 上详细查看 GraphQL 查询有关的内容。

我们的核心价值之一就是“它只是 GraphQL”。当使用 `react-apollo` 时，您不需要学习关于查询语法的任何特殊的东西，因为所有内容都是标准的 GraphQL。您可以在 GraphiQL 查询 IDE 中输入的任何内容，也可以将其添加到 `react-apollo` 代码中。

<h2 id="basics">基本查询</h2>

当我们执行一个基本的查询时，我们可以用一个非常简单的方式使用 `graphql` 容器。我们只需使用 `gql` 模板字符串解析我们的查询，然后将其作为第一个参数传递给 `graphql` 容器。

例如，在 GitHunt 中，我们要在 `Profile` 组件中显示当前登录的用户：

```js
import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

class Profile extends Component { ... }

// 我们使用 gql 标签来将查询字符串解析成查询文档
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

当我们把 GraphQL 查询文档作为参数调用 `graphql` 时，会发生两件事情：

1. 查询是从 Apollo 客户端数据 store 中加载的，如果数据不在 store 中，则向服务端请求；
2. 我们的组件订阅 store 中的数据，以便如果数据因服务端的突变或某些其他响应而发生变化，则会更新该 store 的数据。

除了在查询中选择的 `currentUser` 字段之外，`data` props 还包含一个名为 `loading` 的字段，表示当前是否正在从服务端加载该查询的一个布尔值。所以如果我们要声明 `propTypes`，代码看起来会像这样：

```js
Profile.propTypes = {
  data: PropTypes.shape({
    loading: PropTypes.bool.isRequired,
    currentUser: PropTypes.object,
  }).isRequired,
};
```

随着时间的变化，`data.currentUser` props 将随着客户端获取到的当前用户数据而改变。该信息存储在 Apollo 客户端的全局缓存中，因此如果某些其他查询获取有关当前用户的新信息，则此组件将更新以保持一致。您可以在[关于缓存更新的文章](cache-updates.html)中阅读有关使缓存获取服务器最新数据的技术的相关信息。

<h2 id="default-result-props" title="The data prop">`data` props的结构</h2>

如上所述，`graphql` 将把查询的结果传递给包装组件的一个名为 `data` 的 props 中，它也将传递父容器的所有 props。

对于查询，`data` props 的结构如下所示：

- `...fields`: 查询中每个根字段的一个键。
- `loading`: 如果当前查询还在请求中的状态，包括调用 `refetch` 后，该字段为 `true`，否则为 `false`。
- `error`: 一个 ApolloError 对象，表示执行查询时可能发生的错误。

还有其他更多的方法，您可以阅读关于 [API 文档中查询](api-queries.html＃graphql-query-data)的部分。例如，对于像这样的查询：

```graphql
query getUserAndLikes($id: ID!) {
  user(userId: $id) { name }
  likes(userId: $id) { count }
}
```

你可以得到如下 props：

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

如果你使用 `props` 选项来指定你的子组件的[自定义 `props`](＃graphql-props)，那么这个对象将被作为 `data` 参数传递给 `props` 选项。

<h2 id="graphql-options">变量和选项</h2>

如果要配置查询，可以在 `graphql` 的第二个参数上提供一个包含 `options` 键的对象，您的选项将被传递给 [`ApolloClient.watchQuery`](/core/apollo-client-api.html＃watchQuery)。如果您的查询需要变量，这是传递他们的方法：

```js
// 假设我们的个人资料查询需要指定头像大小
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

<h3 id="options-from-props">根据 props 计算</h3>

通常，查询的变量将从包装组件的 `props` 中计算出来。无论在应用程序中何处使用组件，调用者都会传递参数。所以 `options` 可以是将 props 传递给组件的一个功能：

```js
// 调用者可以执行以下操作：
<ProfileWithData avatarSize={300} />

// 我们的HOC可能看起来像：
const ProfileWithData = graphql(CurrentUserForLayout, {
  options: ({ avatarSize }) => ({ variables: { avatarSize } }),
})(Profile);
```

默认情况下，`graphql` 将尝试从 `ownProps` 的对象中提取任何所需的变量。所以在上面的例子中，我们可以使用更简单的 `graphql(CurrentUserForLayout)(Profile);`。但是，如果您需要更改变量的名称，计算该值，或者只是希望对变量名进行更明确的定义，那么 `options` 函数就是用来这样做的。

<h3 id="other-graphql-options">其他选项</h3>

除了 `variables` 之外，还有很多其他可以传递的选项，例如 `pollInterval`：

```js
const ProfileWithData = graphql(CurrentUserForLayout, {
  // 查看 watchQuery API 以获取您可以在这里提供的选项
  options: { pollInterval: 20000 },
})(Profile);
```

如果您使用函数来计算 props 中的选项，则每当 props 更改时，所有这些 `options` 将自动重新计算。

[阅读API文档中的所有查询选项。](api-queries.html#graphql-query-options)

<h2 id="graphql-skip">跳过操作</h2>

`graphql` 容器API是有意设计成完全静态的，所以你不能在运行时动态地改变查询或包装组件，而不是生成一个新的React组件。但是，有时候，您可能需要执行一些条件判断来根据传入的 props 跳过查询。因此，您可以使用 `skip` 选项。

例如，如果您想忽略未通过身份验证的用户的查询，则可以使用此选项：

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

传递 `skip` 配置值为 true 将完全绕过高阶组件，就好像它根本不存在一样。这也意味着你的子组件根本不会得到一个 `data`  props ，而且 `options` 或 `props` 方法也不会被调用。

<h2 id="graphql-props">控制子 props</h2>

默认情况下，与查询一起使用的 `graphql` 将为包装组件提供一个 `data` props ，其中包含有关查询状态的各种信息。我们还会看到[突变](mutations.html)在 `mutate` prop 上提供回调。光是使用这些默认的名称就足以编写整个应用程序。

但是，如果要将您的 UI 组件与 Apollo 分离，并使其在不同的上下文中可复用，您可能需要修改这些默认 props ，并将其包含在自己的自定义对象和函数中。

<h3 id="graphql-name">更改 prop 名称</h3>

如果要更改默认的 `data` props 的名称，但保持完全相同的结构，可以使用 `graphql` 容器的 `name` 选项。当一个组件通过嵌套的 `graphql` 容器使用多个查询时，将会变得十分有用，其中 `data` props 将被覆盖。

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

// 我们希望prop被称为 'CurrentUserForLayout' 而不是 data
const ProfileWithData = graphql(CurrentUserForLayout, {
  name: 'CurrentUserForLayout'
})(Profile);
```

<h3 id="graphql-props-option">任意转换</h3>

如果想要完全控制子组件的 props，请使用 `props` 选项将查询 `data` 对象映射到传递给子组件的任意数量的 props ：

```js

import React, { Component, PropTypes } from 'react';
import { gql, graphql } from 'react-apollo';

// 在这里，Profile 有一个更通用的 API，它没有耦合到 Apollo 或者我们使用的查询的结构
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
  // ownProps 是由父组件调用时传递到 “ProfileWithData” 的 props 
  props: ({ ownProps, data: { loading, currentUser, refetch } }) => ({
    userLoading: loading,
    user: currentUser,
    refetchUser: refetch,
  }),
})(Profile);
```

这样的使用方式可以实现您的展示组件(`Profile`)和 Apollo 之间的完美解耦。

<h2 id="full-api">完整API</h2>

有关 React Apollo 支持的 GraphQL 查询的所有选项和功能的详细信息，请务必查看[关于 `graphql()` 查询的 API 参考](api-queries.html)。
