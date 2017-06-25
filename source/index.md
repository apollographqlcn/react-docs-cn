---
title: React Apollo
sidebar_title: 简介
description: Apollo Client在您的React应用程序中管理服务器端GraphQL数据，因此您不必亲自管理。
---

这是使用[react-apollo](https://github.com/apollographql/react-apollo)模块在您的React或React Native应用程序中使用GraphQL的官方指南。 React-Apollo是将GraphQL查询绑定到您的React组件的一种方便而强大的方法，因此您可以专注于开发UI，同时数据获取和管理不方便。同时，它拥有所有的钩子和扩展点，使您得到充分的控制。

按照GitHub上的资料库：[React-Apollo](https://github.com/apollographql/react-apollo)，[Apollo Client](https://github.com/apollographql/apollo-client)和[这docs site](https://github.com/apollographql/react-docs)。

<h2 id="tutorials">入门指南</h2>

如果您熟悉Web开发，但以前没有尝试过GraphQL或Apollo，我们已经覆盖了您。这里有一组可以查看的小型教程和示例，在短短几个小时内，您将成为一名专家GraphQL开发人员！或者，您可以直接进入[安装库](initialization.html)。

<h3 id="simple-example"> [1.简单的例子](simple-example.html)</h3>

潜入一个基本的应用程序，显示一个视图与React Native和Apollo。这是您在Apollo Client主页上看到的应用程序，但有一些如何与之进行交互的建议，代码更详细地解释。

<h3 id="full-stack-graphql"> [2.全栈 GraphQL + React 教程](https://dev-blog.apollodata.com/full-stack-react-graphql-tutorial-582ac8d24e3b#.cwvxzphyc)</h3>

本教程将介绍如何设置Apollo Client，如何编写简单的服务器，以及如何将它们连接在一起，而且每个星期几乎都在生产更多的部件。这是由React-Apollo的主要开发者[Jonas Helfer](https://twitter.com/helferjs)编写的。

<h3 id="learn-apollo"> [3.学习阿波罗](https://www.learnapollo.com)</h3>

这个社区开发的教程包括从头到尾构建一个简单的Pokedex应用程序的客户端。它可用于[React](https://www.learnapollo.com/tutorial-react/react-01)，[React Native](https://www.learnapollo.com/tutorial-react-native/react-native-01)等平台。学习Apollo由团队和社区围绕[Graphcool](https://www.graph.cool/)进行维护，这是一个托管的GraphQL后端平台，可让您在没有编写任何代码的情况下站起来。

<h3 id="usage-recipes"> [4.使用和配方](queries.html)</h3>

完成交互式示例和教程后，您可以更深入地深入了解。我们已经尝试编写本指南，以便您可以像书一样阅读它，并发现您可以使用Apollo和GraphQL所做的一切。特别是，查看“使用”部分，了解查询和突变等基本功能，以及有关如何实现更高级目标（如服务器端呈现）的具体方向的“配方”部分。如果您遇到任何问题，请随时向[Apollo标签](http://stackoverflow.com/questions/tagged/apollo)上的[Stack Overflow]或[Apollo社区Slack]([ http://dev.apollodata.com/#slack)！

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
