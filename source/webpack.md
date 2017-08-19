---
title: Webpack加载器
---

你可以使用Webpack在`.graphql`文件上加载GraphQL查询。包`graphql-tag`带有一个容易设置的加载器，并具有一些优点：

1. 不要在客户端处理GraphQL 抽象语法树
2. 将查询与逻辑分开

在下面的例子中，我们创建一个名为`currentUser.graphql`的新文件：

```graphql
query CurrentUserForLayout {
  currentUser {
    login
    avatar_url
  }
}
```

你可以加载此文件在Webpack配置文件中添加规则：

```js
module: {
  rules: [
    {
      test: /\.(graphql|gql)$/,
      exclude: /node_modules/,
      loader: 'graphql-tag/loader',
    },
  ],
},
```

正如你所看到的，`.graphql`或`.gql`文件将在导入时被解析：

```js
import React, { Component } from 'react';
import { graphql } from 'react-apollo';
import currentUserQuery from './currentUser.graphql';

class Profile extends Component { ... }
Profile.propTypes = { ... };

export default graphql(currentUserQuery)(Profile)
```

## Jest

[Jest](https://facebook.github.io/jest/) 不能使用Webpack加载器。要在Jest中进行相同的转换工作，请使用[jest-transform-graphql](https://github.com/remind101/jest-transform-graphql)。

## 片段

你可以在 .graphql 文件中使用并包含片段，并且让webpack包含这些依赖关系，与在纯JS中使用gql标签的片段和查询方式类似。

```graphql
#import "./UserInfoFragment.graphql"

query CurrentUserForLayout {
  currentUser {
    ...UserInfo
  }
}
```

看看我们如何从另一个 .graphql 文件导入UserInfo片段（与在JS中导入模块的方法相同）。

这里是另一个 .graphql 文件中定义片段的示例。

```graphql
fragment UserInfo on User {
  login
  avatar_url
}
```
