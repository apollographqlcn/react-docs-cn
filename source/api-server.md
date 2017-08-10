---
sidebar_title: 服务端渲染
title: "API: 服务端渲染"
---

`react-apollo` 提供了一些实用程序来帮助服务器端呈现和存储水合。 要了解如何在应用程序中实现服务器端呈现，请务必阅读[服务器端渲染的技巧](server-side-rendering.html)。以下仅仅是服务器端渲染中使用的方法的API的参考，而不是教你如何设置的教程。


<h2 id="getDataFromTree" title="getDataFromTree">`getDataFromTree(jsx)`</h2>

```js
import { getDataFromTree } from 'react-apollo';
```

此函数将遍历你的React树，以查找使用 `graphql()` 增强的任何组件。它将采用查询的组件，执行查询，并返回承诺，以便在所有查询已解决时通知你。这个承诺解决了没有价值。你将无法查看找到的查询返回的数据。

当使用[`react-dom/server` 的方法][]像 `renderToString` 或 `renderToStaticMarkup` 渲染时执行`getDataFromTree`后，Apollo缓存将被启动，你的组件将使用缓存中的获取数据进行渲染。你也可以选择使用`react-apollo` [`renderToStringWithData()`](#renderToStringWithData) 方法，该方法将调用此函数，然后调用[`react-dom/server`的 `renderToString`][]。

如果其中一个查询失败，承诺将不会拒绝，直到所有查询都已解决或拒绝。在这一点上，我们将拒绝从 `getDataFromTree` 返回的承诺，其中的错误属性为`error.queryErrors`，它是我们执行的查询中的所有错误的数组。在这一点上，你可能会决定是否渲染树（如果是，错误的组件将处于加载状态），或者渲染错误页面并在客户端上进行完全重新渲染。

有关更多信息，请参阅[服务端渲染的技巧](server-side-rendering.html)。

[`react-dom/server` 的方法]: https://facebook.github.io/react/docs/react-dom-server.html
[`react-dom/server`的 `renderToString`]: https://facebook.github.io/react/docs/react-dom-server.html#rendertostring

**例：**

```js
const jsx = (
  <ApolloProvider client={client}>
    <MyRootComponent/>
  </ApolloProvider>
);

getDataFromTree(jsx)
  .then(() => {
    console.log(renderToString(jsx));
  })
  .catch((error) => {
    console.error(error);
  });
```

<h2 id="renderToStringWithData" title="renderToStringWithData">
  `renderToStringWithData(jsx)`
</h2>

```js
import { renderToStringWithData } from 'react-apollo';
```

这个函数调用[`getDataFromTree()`](#getDataFromTree)，当该函数返回的承诺解析它调用[`react-dom/server`’s `renderToString`][]。

有关更多信息，请参阅[`getDataFromTree()`](#getDataFromTree)或[服务器端渲染的技巧](server-side-rendering.html)的文档。

[`react-dom/server` 的 `renderToString`]: https://facebook.github.io/react/docs/react-dom-server.html#rendertostring

**例：**

```js
renderToStringWithData(
  <ApolloProvider client={client}>
    <MyRootComponent/>
  </ApolloProvider>
)
  .then((html) => {
    console.log(html);
  })
  .catch((error) => {
    console.error(error);
  });
```
