---
title: React Apollo
sidebar_title: 简介
description: Apollo Client 会管理好 React 应用的后台 GraphQ L 数据，不用你自己操心。
---

这是一份官方指南，指导您如何在 React 或 React Native 应用中借助 [react-apollo](https://github.com/apollographql/react-apollo) 来使用 Graphql。通过 React-Apollo 将 GraphQL 查询绑定到 React 组件，这种方式既简便又强大，因为它帮你做好数据请求和管理，然后你就可以专心开发 UI。同时，它提供丰富的钩子方法和扩展机制，让您完全可控。

关注 GitHub 上的仓库：[React-Apollo](https://github.com/apollographql/react-apollo)，[Apollo Client](https://github.com/apollographql/apollo-client) 和 [这docs site](https://github.com/apollographql/react-docs)。

<h2 id="tutorials">快速入门</h2>

如果你熟悉 Web 开发，但以前没有用过 GraphQL 或 Apollo，不怕，这里有一套简明教程和例子，花几个小时，你就可以成为高级 GraphQL 开发了！如果你熟悉 Graphql，则可以直接进入[安装](initialization.html)。

<h3 id="simple-example"> [1.简单的例子](simple-example.html)</h3>

快速进入一个简单 app，展示一个用 React Native 与 Apollo 开发的界面。这就是在 Apollo Client 主页上看到的 app，不过多了一些操作交互的建议，还有更详细的代码解释。

<h3 id="full-stack-graphql"> [2.全栈 GraphQL + React 教程](https://dev-blog.apollodata.com/full-stack-react-graphql-tutorial-582ac8d24e3b#.cwvxzphyc)</h3>

本教程将介绍如何搭建 Apollo Client、编写简单服务器，以及如何将它们连接在一起，这个教程几乎每周都在更新。这是由 React-Apollo 的核心开发者 [Jonas Helfer](https://twitter.com/helferjs) 编写的。

<h3 id="learn-apollo"> [3.学习 Apollo](https://www.learnapollo.com)</h3>

这份社区发起的教程，指导你从零开始构建一个简单的 宠物小精灵 前端应用。有 [React](https://www.learnapollo.com/tutorial-react/react-01) 版，[React Native](https://www.learnapollo.com/tutorial-react-native/react-native-01) 版 和其他平台版本。这个「学习 Apollo」教程是由 [Graphcool](https://www.graph.cool/) 团队和社区进行维护，还提供一个托管的 GraphQL 后端平台，让你不用写一行代码就可以跑 GraphQl 服务器。

<h3 id="usage-recipes"> [4. 使用与实践](queries.html)</h3>

学习完例子和教程，就可以更加深入了。我们花了很多时间编写这份指南，让你可以像看书一样学习一切 Apollo 和 GraphQL 能做的。特别是，查看「使用」部分，了解 querys 和 mutations 等基本功能，而「实践」部分则用于一些实现更高级目标的特定场景，例如服务端渲染。如果您遇到任何问题，随时向 [Stack Overflow (带上 `apollo` 标签)](http://stackoverflow.com/questions/tagged/apollo) 或 [Apollo 社区 Slack]([ http://dev.apollodata.com/#slack) 提问！

<h2 id="compatibility">兼容工具</h2>

Apollo Client 的首要设计原则就是兼容你正在使用的前后端工具。维护人员关注解决这些难题：GraphQL 缓存、请求管理、UI 更新，如果对其他方面有任何技术需求或者不同偏好，我们希望也能做到位。

<h3 id="react-toolbox">React 工具</h3>

Apollo 经过专门设计，可以跟目前 React 开发者使用的工具很好的结合。下面列举一些：

- **React Native 和 Expo**：Apollo 在 React Native 中开箱即用。它甚至预装在 [Expo Sketch](https://sketch.expo.io/H1QdWZUjg) 中，因此你可以在浏览器中直接构建一个 React Native + Apollo 的应用。

- ** Redux **：Apollo Client 内部使用 Redux，你可以将其[集成到现有 store](redux.html)，使用你喜欢的 Redux 工具，例如 dev tools 或 数据存储持久库。你还可以将 Apollo 与任何其他数据管理库（如 MobX）一起使用。

- ** React Router **：Apollo Client 完全独立于路由，这意味着你可以使用任何版本的 [React Router](https://github.com/ReactTraining/react-router) 或其他适合 React 的路由库。另外搭建[服务器端渲染](http://dev.apollodata.com/react/server-side-rendering.html)也特别简单。

- **Recompose** 借助 [Recompose](https://github.com/acdlite/recompose)，React-Apollo 的高阶组件可以与各种工具结合，为组件增强功能。 读读[如何使用它来加载状态和变量](https://dev-blog.apollodata.com/simplify-your-react-components-with-apollo-and-recompose-8b9e302dea51#.z7tbkf8er)、[mutations](https://medium.com/front-end-developers/how-i-write-mutations-in-apollo-w-recompose-1c0ab06ef4ea#.iobufopba) 以及[与 Redux container 相结合](https：/ /medium.com/welikegraphql/use-of-recompose-in-universal-react-apollo-example-3d1f89bc945b#.dtxnibu0w)。

- ** Next.js **：您可以结合轻量级的 Next.js 框架 使用 Apollo，实现同构渲染 React 应用。有关详细信息，请查看 [这篇文章](https://dev-blog.apollodata.com/whats-next-js-for-apollo-e4dfe835d070)，或下载 [官方示例](https://github.com/zeit/next.js/tree/master/examples/with-apollo)。

如果你有一个喜欢的 React 工具，但是集成到 Apollo  有些难度，请提一个 issue，我们一起让它好地集成起来。

<h3 id="graphql-servers"> GraphQL 服务器</h3>

因为 Apollo 不会假定任超出何官方 GraphQL 规范以外的东西，所以它可以应用在各种语言实现的 GraphQL 服务器。它也不对你的 schema 作额外的要求;如果你可以向符合规范的 GraphQL 服务器发送查询，那么 Apollo 就可以处理它。你可以在[graphql.org](http://graphql.org/code/#server-libraries)上找到一些 GraphQL 服务器的实现。

<h3 id="other-platforms">其他 JS、native平台</h3>

这个文档网站是特别针对 React的，但 Apollo 已为每个客户端平台提供了实现：

- JavaScript
   - [Angular](/angular)
   - [Vanilla JS](/core)
- 原生移动平台
   - [原生 iOS (Swift)](http://dev.apollodata.com/ios/)
   - [原生 Android (Java)](https://github.com/apollographql/apollo-android)

<h2 id="goals">项目目标</h2>

Apollo Client 是 GraphQL 的 JavaScript 客户端。我们让 Apollo Client 做到：

1. **增量可用**，你可以在已有的 JavaScript 应用中，对于某一部分 UI 使用 GraphQL。
2. **通用兼容**，Apollo 可以接受各种不同构建运行的 GraphQL 服务器，以及任何 GraphQL schema。
//TODO HERE
3. **简单的开始使用**，你可以阅读一个小的教程，并开始。
** **可检查和可理解**，以便您可以拥有优秀的开发人员工具来准确了解应用程式中发生的情况。
4. **专为交互式应用程序**而设计，因此您的用户可以立即进行更改并将其反映在UI中。
**社区驱动**，让您有信心，该项目将随着您的需求而增长。 Apollo软件包从一开始就与生产用户共同开发，所有项目都在GitHub开放计划和开发，所以没有任何惊喜。

Apollo Client不仅仅是简单地对您的GraphQL服务器运行查询。它分析您的查询及其结果以构建数据的客户端缓存，随着进一步的查询和突变的运行，该数据将保持最新。这意味着您的UI可以内部一致，并且与服务器上的状态完全同步，并且需要最少的查询数量。

<h2 id="comparison">其他GraphQL客户端</h2>

如果您决定是否使用“react-apollo”或其他GraphQL客户端，那么值得考虑的是项目的[目标](＃个目标)以及它们的比较。这里还有一些要点：

  - [Relay](https://facebook.github.io/relay/)是由Facebook为其移动应用程序构建的面向性能和高度评价的GraphQL客户端。它专注于实现查询和组件的共同定位，并且对于GraphQL模式的设计有所了解，特别是在分页的情况下。 Apollo具有与Relay相似的一组功能，但它被设计为可用于任何模式或任何前端架构的通用工具。继电器与特定架构的耦合可以带来一些好处，但是失去了一些灵活性，这也使得阿波罗社区更快速地进行迭代并快速测试实验特性。
  - [Lokka](https://github.com/kadirahq/lokka)是一个简单的GraphQL JavaScript客户端，具有基本的查询缓存。阿波罗更复杂，但包括更复杂的缓存和一系列高级功能，用于更新和重新获取数据。

<h2 id="learn-more”>其他资源</h2>

- [GraphQL.org](http://graphql.org)，介绍和引用GraphQL本身，由Apollo团队部分编写和维护。
- [我们的网站](http://www.apollodata.com/)了解阿波罗的开源和商业工具。
- [Medium 专栏博客](https://medium.com/apollo-stack)，关于GraphQL的长篇文章，“阿波罗”的功能公告，以及来自社区的访客文章。
- [我们的Twitter](https://twitter.com/apollographql)为即时消息。
