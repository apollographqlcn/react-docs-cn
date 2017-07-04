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

这份社区发起的教程，知道你从零开始构建一个简单的 宠物小精灵 前端应用。有 [React](https://www.learnapollo.com/tutorial-react/react-01) 版，[React Native](https://www.learnapollo.com/tutorial-react-native/react-native-01) 版 和其他平台版本。这个「学习 Apollo」教程是由 [Graphcool](https://www.graph.cool/) 团队和社区进行维护，还提供一个托管的 GraphQL 后端平台，让你不用写一行代码就可以跑 GraphQl 服务器。

<h3 id="usage-recipes"> [4. 使用与实践](queries.html)</h3>

学习完例子和教程，就可以更加深入了。我们花了很多时间编写这份指南，让你可以像看书一样学习一切 Apollo 和 GraphQL 能做的。特别是，查看「使用」部分，了解 querys 和 mutations 等基本功能，而「实践」部分则用于一些实现更高级目标的特定场景，例如服务端渲染。如果您遇到任何问题，随时向 [Stack Overflow (带上 `apollo` 标签)](http://stackoverflow.com/questions/tagged/apollo) 或 [Apollo 社区 Slack]([ http://dev.apollodata.com/#slack) 提问！

<h2 id="compatibility">兼容工具</h2>

Apollo Client的主要设计点是使用您已经在使用的客户端或服务器工具。维护者和贡献者专注于解决GraphQL缓存，请求管理和UI更新周围的困难问题，我们希望这些可用于任何人，无论他们对应用程序的其他部分的技术要求和偏好。

<h3 id="react-toolbox">React 工具</h3>

Apollo专门设计用于与当今React开发人员使用的所有工具配合使用。以下是一些特别的：

- **反应本土和世博会**：阿波罗在React Native中开箱即用。它甚至预装在[Expo Sketch](https://sketch.expo.io/H1QdWZUjg)中，因此您可以在浏览器中直接构建一个React Native + Apollo应用程序。
- ** Redux **：Apollo Client在内部使用Redux，您可以[将其集成到现有存储中](redux.html)以使用您最喜欢的Redux工具，例如开发工具或持久性库。您还可以将Apollo与任何其他数据管理库（如MobX）一起使用。
- ** React Router **：Apollo Client完全与路由器无关，这意味着您可以使用任何版本的[React Router](https://github.com/ReactTraining/react-router)或任何其他路由库为反应。甚至可以设置[服务器端渲染](http://dev.apollodata.com/react/server-side-rendering.html)。
- **重新组合**：使用[Recompose](https://github.com/acdlite/recompose），React-Apollo的高阶组件可以与各种其他实用程序相结合，为组件添加行为。 [阅读如何使用它来加载状态和变量](https://dev-blog.apollodata.com/simplify-your-react-components-with-apollo-and-recompose-8b9e302dea51#.z7tbkf8er)以及[突变](https://medium.com/front-end-developers/how-i-write-mutations-in-apollo-w-recompose-1c0ab06ef4ea#.iobufopba）和[与Redux容器相结合](https：/ /medium.com/welikegraphql/use-of-recompose-in-universal-react-apollo-example-3d1f89bc945b#.dtxnibu0w)。
- ** Next.js **：您可以使用Apollo与轻量级的Next.js框架进行通用渲染的React应用程序。有关详细信息，请查看[本文](https://dev-blog.apollodata.com/whats-next-js-for-apollo-e4dfe835d070)，或下载[官方示例](https：// github。 COM /宰特/ next.js /树/主/示例/与-阿波罗)。

如果您有一个最喜欢的React工具，并且Apollo中的某些东西使得难以集成，请打开一个问题，让我们一起工作，使其正常工作并将其添加到列表中！

<h3 id="graphql-servers"> GraphQL服务器</h3>

因为它不承担任何官方GraphQL规范以外的任何东西，所以Apollo可以针对每种语言使用每个GraphQL服务器实现。它也不对您的模式施加任何要求;如果您可以向标准GraphQL服务器发送查询，则Apollo可以处理它。您可以在[graphql.org](http://graphql.org/code/#server-libraries)上找到GraphQL服务器实现的列表。

<h3 id="other-platforms">其他JS +本地平台</h3>

本文档网站特别针对React，但Apollo已为每个客户端平台实施：

- JavaScript
   - [Angular](/angular)
   - [Vanilla JS](/core)
- 原生移动平台
   - [Native iOS with Swift](http://dev.apollodata.com/ios/)
   - [Native Android with Java](https://github.com/apollographql/apollo-android)

<h2 id="goals">项目目标</h2>

Apollo Client是GraphQL的JavaScript客户端。我们建立了Apollo客户端：

1. **增量可以采用**，以便您可以将其放入现有的JavaScript应用程序中，并开始使用GraphQL作为UI的一部分。
2. **通用兼容**，因此Apollo可以使用任何构建设置，任何GraphQL服务器和任何GraphQL模式。
** **简单的开始使用**，你可以阅读一个小的教程，并开始。
** **可检查和可理解**，以便您可以拥有优秀的开发人员工具来准确了解应用程式中发生的情况。
** **专为交互式应用程序**而设计，因此您的用户可以立即进行更改并将其反映在UI中。
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
