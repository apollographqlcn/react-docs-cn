---
title: 与 React Native 集成
---

你可以使用Apollo与React Native，就像使用React Web一样。

要将Apollo引入你的应用程序，请从npm安装 `react-apollo`，并在[设置](initialization.html)文章中概述的应用程序中使用它们：

```bash
npm install react-apollo --save
```

```js
import React from 'react';
import { AppRegistry } from 'react-native';
import { ApolloClient, ApolloProvider } from 'react-apollo';

// 如上所述创建客户端
const client = new ApolloClient();

const App = () => (
  <ApolloProvider client={client}>
    <MyRootComponent />
  </ApolloProvider>
);

AppRegistry.registerComponent('MyApplication', () => App);
```

如果你刚开始使用 Apollo with React，你应该阅读[React guide](index.html)。

<h2 id="examples">示例</h2>

有一些Appollo的例子写在React Native中，你可能希望参考：

1. dev.apollodata.com 使用的[“Hello World”示例](https://github.com/apollographql/frontpage-react-native-app)。
2. 一个[GitHub API示例](https://github.com/apollographql/GitHub-GraphQL-API-Example)构建为与GitHub的新GraphQL API配合使用。

> 如果你有一个例子来发布在这里，请点击上面的“编辑GitHub”按钮，让我们知道！
