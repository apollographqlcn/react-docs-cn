---
title: 服务端渲染
---

Apollo 提供了两种技术，可让你的应用快速加载，为用户避免不必要的延迟：

- Store 覆水，它将允许你的初始查询集立即返回数据，而无需服务端请求。
- 服务端渲染，它将服务端渲染好的初始 HTML 视图发送给客户端。

你可以使用这些技术中的一种或两种来提供更好的用户体验。

<h2 id="store-rehydration">Store 覆水</h2>

对于能在客户端渲染 UI 之前于服务端执行查询的应用，Apollo 运行你为其设置数据的初始状态。这有时被称为覆水，因为数据被序列化并被包含在初始 HTML 代码中时，称数据被“脱水”。

例如，一个典型的方法是包括一个脚本标签，如下：

```html
<script>
  // `initialState` 应该具有 Apollo store 状态的形状。确保只包括必要数据，例如：
  // const initialState = {[client.reduxRootKey]: {
  //   data: client.store.getState()[client.reduxRootKey].data
  // }};
  window.__APOLLO_STATE__ = initialState;
</script>
```

然后可以使用从服务器传来的初始状态来为客户端执行覆水操作：

```js
const client = new ApolloClient({
  initialState: window.__APOLLO_STATE__,
});
```

我们将在下面演示如何使用 Node 和 `react-apollo` 的服务端渲染功能生成 HTML 和 Apollo store 的状态。但是，如果你通过其他方式渲染 HTML，则必须手动生成该状态。

如果你在 Apollo 外部使用 [Redux](redux.html)，并且已经为 store 覆水，则应将 store 状态传递到 [`Store` 构造函数](http://redux.js.org/docs/basics/Store.html)。

然后，当客户端运行第一组查询时，数据将立即返回，因为它已经在 store 中！

如果你在某些初始查询中使用 [`forceFetch`](cache-updates.html#forceFetch)，则可以在初始化期间传递 `ssrForceFetchDelay` 选项来跳过强制获取，以便即使是这些查询也将使用缓存的数据源：

```js
const client = new ApolloClient({
  initialState: window.__APOLLO_STATE__,
  ssrForceFetchDelay: 100,
});
```

<h2 id="server-rendering">服务端渲染</h2>

你可以使用 `react-apollo` 中内置的渲染函数在 Node 服务器上渲染整个基于 React 的 Apollo 应用。这些函数负责获取渲染组件树所需的所有查询。通常，你可以在 HTTP 服务器（如 [Express](https://expressjs.com)）中使用这些功能。

不需要更改客户端查询来支持服务端渲染，因此你的基于 Apollo 的 React UI 支持开箱即用的 SSR。

<h3 id="server-initialization">服务端初始化</h3>

In order to render your application on the server, you need to handle a HTTP request (using a server like Express, and a server-capable Router like React-Router), and then render your application to a string to pass back on the response.

We’ll see how to take your component tree and turn it into a string in the next section, but you’ll need to be a little careful in how you construct your Apollo Client instance on the server to ensure everything works there as well:

When creating an Apollo Client instance on the server, you’ll need to set up you network interface to connect to the API server correctly. This might look different to how you do it on the client, since you’ll probably have to use an absolute URL to the server if you were using a relative URL on the client.
Since you only want to fetch each query result once, pass the ssrMode: true option to the Apollo Client constructor to avoid repeated force-fetching.
You need to ensure that you create a new client or store instance for each request, rather than re-using the same client for multiple requests. Otherwise the UI will be getting stale data and you’ll have problems with authentication.
Once you put that all together, you’ll end up with initialization code that looks like this:

为了在服务器上渲染应用程序，你需要处理 HTTP 请求（使用像 Express 这样的服务端框架和支持服务端运行的路由库，如 React-Router），然后将应用渲染为一个字符串以传回响应。

我们将在下一部分中看到如何使用组件树并将其转换为一个字符串，但是你需要对如何在服务器上构建 Apollo 客户端实例保持谨慎，以确保一切都能工作完好：

1. 当在服务器上[创建 Apollo 客户端实例](initialization.html)时，需要设置网络接口才能正确连接到 API 服务器。这可能与你在客户端上的操作看起来有所不同，因为如果你在客户端上使用相对 URL，则可能需要在服务端使用绝对 URL。

2. 由于你只需获取每个查询的结果一次，请将 `ssrMode: true` 选项传递给 Apollo 客户端构造函数，以避免重复强制请求。

3. 你需要确保为每个请求创建一个新的客户端或 store 实例，而不是为多个请求重复使用同一个客户端。否则，用户界面将变得过时，而且还会遇到[认证](auth.html)的问题。

一旦你把它们放在一起，你会得到如下初始化代码：

```js
// This example uses React Router v4, although it should work
// equally well with other routers that support SSR

import { ApolloClient, createNetworkInterface, ApolloProvider } from 'react-apollo';
import Express from 'express';
import { match, RouterContext } from 'react-router';

// 路由文件是客户端和服务器之间良好的共享入口点
import routes from './routes';

// 请注意，你不必使用任何特定的http服务器，但是我们在这个例子中使用Express
const app = new Express();
app.use((req, res) => {

  // 此示例使用React Router，尽管它应该与支持SSR的其他路由器同样好
  match({ routes, location: req.originalUrl }, (error, redirectLocation, renderProps) => {

    const client = new ApolloClient({
      ssrMode: true,
      // 请记住，这是SSR服务器将用于连接到API服务器的接口，因此我们需要确保它不被防火墙等
      networkInterface: createNetworkInterface({
        uri: 'http://localhost:3010',
        opts: {
          credentials: 'same-origin',
          headers: {
            cookie: req.header('Cookie'),
          },
        },
      }),
    });

    const app = (
      <ApolloProvider client={client}>
        <RouterContext {...renderProps} />
      </ApolloProvider>
    );

    // 渲染代码（见下文）
  });
});

app.listen(basePort, () => console.log( // eslint-disable-line no-console
  `App Server is now running on http://localhost:${basePort}`
));
```

你可以查看[ GitHunt 应用的 `ui/server.js`](https://github.com/apollographql/GitHunt-React/blob/master/ui/server.js) 以获取完整的代码示例。

接下来我们将看看渲染代码实际上是什么。

<h3 id="getDataFromTree">使用 `getDataFromTree`</h3>

`getDataFromTree` 方法使用 React 树，确定需要哪些查询来渲染它们，然后将其全部获取。如果你有嵌套查询，它会递归地放在整个树上。它返回一个 promise，当你的Apollo 客户端 store 中的数据准备就绪时可以被解析。

在 promise 解析之前，你的 Apollo 客户端 store 将被完全初始化，这意味着你的应用将立即呈现（因为所有查询都被预取），并且你可以在响应中返回字符串结果：

```js
import { getDataFromTree } from "react-apollo"

const client = new ApolloClient(....);

// 请求（见上文）
getDataFromTree(app).then(() => {
  // 我们准备好渲染真实的DOM
  const content = ReactDOM.renderToString(app);
  const initialState = {[client.reduxRootKey]: client.getInitialState()  };

  const html = <Html content={content} state={initialState} />;

  res.status(200);
  res.send(`<!doctype html>\n${ReactDOM.renderToStaticMarkup(html)}`);
  res.end();
});
```

在这种情况下，你的 HTML 可能看起来像这样：

```js
function Html({ content, state }) {
  return (
    <html>
      <body>
        <div id="content" dangerouslySetInnerHTML={{ __html: content }} />
        <script dangerouslySetInnerHTML={{
          __html: `window.__APOLLO_STATE__=${JSON.stringify(state).replace(/</g, '\\u003c')};`,
        }} />
      </body>
    </html>
  );
}
```

<h3 id="local-queries">避免本地查询的使用外网</h3>

如果你的 GraphQL 节点与你做服务端渲染的是在同一台服务器上，则可能希望在进行 SSR 查询时避免使用外网。特别是，如果本地主机在你的生产环境（例如 Heroku ）上配置了防火墙，则这些查询的网络请求将不起作用。该问题的一个解决方案是 [apollo-local-query](https://github.com/af/apollo-local-query) 模块，它允许你为 Apollo 创建一个实际不使用外网的 `networkInterface`。

<h3 id="skip-for-ssr">跳过SSR查询</h3>

如果要在 SSR 期间有意地跳过查询，可以在查询选项中设置 `ssr: false`。通常，这将意味着该组件在服务器上将处于加载状态。例如：

```js
const withClientOnlyUser = graphql(GET_USER_WITH_ID, {
  options: { ssr: false }, // 不会在 SSR 期间调用
});
```

<h3 id="renderToStringWithData">使用 `renderToStringWithData`</h3>

`renderToStringWithData` 方法简化了上面的内容，只需返回需要渲染的内容字符串即可。所以它略微减少了步骤：

```js
// 服务端应用代码（集成使用）
import { renderToStringWithData } from "react-apollo"

const client = new ApolloClient(....);

// 在请求期间
renderToStringWithData(app).then((content) => {
  const initialState = {[client.reduxRootKey]: client.getInitialState() };
  const html = <Html content={content} state={initialState} />;

  res.status(200);
  res.send(`<!doctype html>\n${ReactDOM.renderToStaticMarkup(html)}`);
  res.end();
});
```
