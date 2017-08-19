---
sidebar_title: 服务端渲染
title: "API: 服务端渲染"
---

`react-apollo` 提供了一些实用的方法来帮助服务端渲染和 store 注水。要了解如何在应用中实现服务端渲染，请务必阅读[服务端渲染的方法](server-side-rendering.html)。以下仅仅是服务端渲染中所使用的方法的 API 参考，而不是教你如何设置的教程。


<h2 id="getDataFromTree" title="getDataFromTree">`getDataFromTree(jsx)`</h2>

```js
import { getDataFromTree } from 'react-apollo';
```

此函数将遍历您的 React 树，以查找任何使用 `graphql()` 增强器的组件。它将找到那些查询组件，执行查询，并返回一个 promise，以便在所有查询已 resolve 时通知您。这个 promise 不会返回任何值。您将无法查看找到的查询返回的数据。

当使用 [`react-dom/server` 的方法][]如 `renderToString` 或 `renderToStaticMarkup` 渲染模板时，在执行 `getDataFromTree` 后，Apollo 缓存将被启动，您的组件将使用缓存中已获取的数据进行渲染。您也可以选择使用 `react-apollo` [`renderToStringWithData()`](#renderToStringWithData) 方法，该方法将调用此函数，然后调用 [`react-dom/server`的 `renderToString`][] 方法。

如果其中一个查询失败，promise 将不会 reject，直到所有查询都已 resolve 或 reject。这时，我们将使用一个属性为 `error.queryErrors` 的错误对象 reject 从 `getDataFromTree` 返回的 promise，该属性是我们执行的查询中所有错误的数组。这时，您再决定是否渲染树（如果是，错误的组件将处于加载状态），或者渲染一个错误页面并在客户端上完全重新渲染。

有关更多信息，请参阅[服务端渲染的方法](server-side-rendering.html)。

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

这个函数会调用 [`getDataFromTree()`](#getDataFromTree)，当该函数返回的 promise 被 resolve 时，它将调用[`react-dom/server`的 `renderToString`][] 方法。

有关更多信息，请参阅 [`getDataFromTree()`](#getDataFromTree) 或 [服务器端渲染的方法](server-side-rendering.html)文档。

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
