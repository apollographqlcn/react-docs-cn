---
title: "完整示例：GitHunt"
---
GitHunt is an application in the style of Product Hunt that shows a list of GitHub repositories, sorted by votes with attached comments. It can be a useful reference to see a lot of Apollo features in action, although it has a lot of functionality so the code is not simple for beginners.

View the live app frontend
Run queries against the live server

GitHunt是一种以产品搜寻为特色的应用程序，它显示了GitHub存储库的列表，按投票和附加注释排序。 尽管它具有很多功能，但代码对于初学者来说并不简单，它可以帮助你了解很多Apollo功能。

- [查看现场应用程序前端](http://www.githunt.com/)
- [针对实时服务器运行查询](http://api.githunt.com/graphiql)

[![GitHunt Screenshot](img/githunt.png)](http://www.githunt.com/)

<h2 id="code">获取代码</h2>
Get the code

The API server code demonstrates combining two data sources–a third-party API and a local database–in a single GraphQL endpoint.
The React UI demonstrates a lot of the concepts in this guide.
- [API服务器代码](https://github.com/apollographql/GitHunt-API)演示了在单个GraphQL端点中组合两个数据源（第三方API和本地数据库）。
- [React UI](https://github.com/apollographql/GitHunt-React) 演示了本指南中的许多概念。
