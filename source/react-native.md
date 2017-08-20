---
title: 与 React Native 集成
---

你可以在 React Native 中使用 Apollo，就像 React Web 一样。

要将 Apollo 引入你的应用，请从 npm 安装 `react-apollo`，并按照[安装和配置](initialization.html)文档中所述方式，在你的应用中使用它们：

```bash
npm install react-apollo --save
```

```js
import React from 'react';
import { AppRegistry } from 'react-native';
import { ApolloClient, ApolloProvider } from 'react-apollo';

// 如上所述创建客户端实例
const client = new ApolloClient();

const App = () => (
  <ApolloProvider client={client}>
    <MyRootComponent />
  </ApolloProvider>
);

AppRegistry.registerComponent('MyApplication', () => App);
```

如果你第一次在 React 中使用 Apollo，你应该先阅读 [React 指南](index.html)。

<h2 id="examples">示例</h2>

这有一些你可能希望参考的在 React Native 中 使用 Appollo 的例子：

1. dev.apollodata.com 使用的 [“Hello World” 示例](https://github.com/apollographql/frontpage-react-native-app)。
2. 一个为与 GitHub 的新 GraphQL API 配合使用而构建的 [GitHub API 示例](https://github.com/apollographql/GitHub-GraphQL-API-Example)。

> 如果你想要在这里发布一个示例，请点击上面的“在 GitHub 上编辑本文”按钮，让我们知道！
