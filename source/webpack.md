---
title: Webpack 加载器
---

你可以使用 Webpack 加载 `.graphql` 文件里的 GraphQL 查询语句。`graphql-tag` 包自带一个易于配置的加载器，它具有如下优点：

1. 不在客户端处理 GraphQL 抽象语法树
2. 将查询与逻辑分开

在下面的例子中，我们创建一个名为 `currentUser.graphql` 的新文件：

```graphql
query CurrentUserForLayout {
  currentUser {
    login
    avatar_url
  }
}
```

你可以通过在 Webpack 配置文件中添加规则来加载此文件：

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

正如你所见，`.graphql` 或 `.gql` 文件将在导入时被解析：

```js
import React, { Component } from 'react';
import { graphql } from 'react-apollo';
import currentUserQuery from './currentUser.graphql';

class Profile extends Component { ... }
Profile.propTypes = { ... };

export default graphql(currentUserQuery)(Profile)
```

## Jest

[Jest](https://facebook.github.io/jest/) 不能使用 Webpack 加载器。要在 Jest 中进行相同的转换工作，请使用 [jest-transform-graphql](https://github.com/remind101/jest-transform-graphql)。

## FuseBox

[FuseBox](http://fuse-box.org) 不能使用 Webpack 加载器。要在 FuseBox 中进行相同的转换工作，请使用 [fuse-box-graphql-plugin](https://github.com/otothea/fuse-box-graphql-plugin).

## 片段

你可以在 .graphql 文件中使用并包含片段，并且让 webpack 为你处理这些依赖关系，这与你在普通 JS 中使用 gql tag 的片段和查询方式类似。

```graphql
#import "./UserInfoFragment.graphql"

query CurrentUserForLayout {
  currentUser {
    ...UserInfo
  }
}
```

看看我们如何从一个 .graphql 文件导入 UserInfo 片段（这与你在 JS 中导入模块的方法相同）。

这是在另一个 .graphql 文件中定义片段的示例：

```graphql
fragment UserInfo on User {
  login
  avatar_url
}
```
