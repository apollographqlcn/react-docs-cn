---
title: 简易示例：Snack
description: 一个小应用，实现了 GitHunt 示例应用中的一个视图，你可以直接从浏览器端运行和编辑。
---

想要知道 Apollo 客户端和 GraphQL 能为你做什么，最简单的方法是自己尝试一番。以下是单个 React Native 视图的简单示例，它使用 Apollo 客户端与我们的托管示例应用 [GitHunt](example-schema.html) 进行通信。我们已经把 [Expo](https://expo.io) 中的 [Snack](https://blog.expo.io/sketch-a-playground-forrereate-native-16b2401f44a2) 编辑器的页面嵌入了本页。

<div data-snack-id="HkhGxRFhe" data-snack-platform="ios" data-snack-preview="true" style="overflow:hidden;background:#fafafa;border:1px solid rgba(0,0,0,.16);border-radius:4px;height:514px;width:100%"></div>
<script async src="https://snack.expo.io/embed.js"></script>

首先，让我们来运行该应用。有两种方法：

1. 在模拟器中点击 "Tap to Play" 按钮运行此应用。
2. 在[新窗口](https://snack.expo.io/HkhGxRFhe)中打开编辑器，并安装 [Expo app](https://expo.io/) 以在 iOS 或 Android 设备上运行。

无论哪种方式，应用都将在你修改代码时自动重新加载。

<h2 id="first-edit">首次编辑</h2>

这个应用是为了让我们练习而特意设置的，因此，在启动应用后，你应该能看到一个展示一些 GitHub 仓库的可滚动列表视图。这些数据是使用 GraphQL 查询从服务端获取的。我们通过移除 `stargazers_count` 前面的 `#` 来编辑查询，这样应用也会加载 GitHub 的 star 数。查询现在应如下所示：

```graphql
{
  feed (type: TOP, limit: 10) {
    repository {
      name, owner { login }

      # 取消下一行的注释以获取 star！
      stargazers_count
    }

    postedBy { login }
  }
}
```

因为在本例中，我们已经开发了相应的 UI，它知道如何显示这些新信息，你应该会看到应用刷新，并显示每个仓库中的 GitHub star 的数量！

<h2 id="github-api">多个后端</h2>

GraphQL 最酷的一点是它可以作为多个后端之上的抽象层。你上面执行的查询实际上是从两个完全独立的数据源一次加载的：一个存储提交列表的 PostgreSQL 数据库，以及用于获取相关仓库信息的 GitHub REST API。

让我们验证一下该应用是_真的_从 GitHub 读取。选择列表中的一个存储库，例如 [apollographql/apollo-client](https://github.com/apollographql/apollo-client) 或 [facebook/graphql](https://github.com/facebook/graphql)，并点下 star。然后，在应用中下拉刷新列表。你应该会看到 star 数字改变了！这是因为我们的 GraphQL 服务端每次都从真实的 GitHub API 中提取数据，并通过一些优化的缓存和 ETag 来控制避免触发请求频率的限制。

<h2 id="code-explanation">代码详解</h2>

在你开始使用 Apollo 构建自个儿棒棒的 GraphQL 应用之前，让我们先来看看这个简单示例中的代码。

这一段代码使用 `react-apollo` 中的 `graphql` 高阶组件将 GraphQL 查询结果附加到 `Feed` 组件：

```js
const FeedWithData = graphql(gql`{
  feed (type: TOP, limit: 10) {
    repository {
      name, owner { login }

      # 取消下一行的注释以获取 star！
      # stargazers_count
    }

    postedBy { login }
  }
}`, { options: { notifyOnNetworkStatusChange: true } })(Feed);
```

这是 React Native 渲染的主要 `App` 组件。它创建了一个带有服务器 URL 的网络接口，初始化一个 `ApolloClient` 实例，并使用 `ApolloProvider` 将其附加到我们的 React 组件树。如果你用过 Redux，这对你来说会很熟悉，因为它与 Redux provider 的工作原理相类似。

```js
export default class App {
  createClient() {
    // 使用我们的服务器 URL 初始化 Apollo 客户端
    return new ApolloClient({
      networkInterface: createNetworkInterface({
        uri: 'http://api.githunt.com/graphql',
      }),
    });
  }

  render() {
    return (
      // 将客户端实例引入 React 组件树中
      <ApolloProvider client={this.createClient()}>
        <FeedWithData />
      </ApolloProvider>
    );
  }
}
```

接下来，我们分析一个实际处理数据加载的组件。大多数情况下，只需通过 `data` prop 把数据传递给 `feedList`，它就会展示出这些项目。但是，这里还有一个有趣的 React Native `RefreshControl` 组件，当你下拉列表时，它使用 Apollo 内置的 `data.refetch` 方法来提取数据。它还使用了 `data.networkStatus` 属性显示正确的加载状态，该状态由 Apollo 为我们跟踪处理。

```js
// 这里的数据来源于 Apollo 高阶组件。它具有我们所需的数据，还有一些有用的方法，如 refetch().
function Feed({ data }) {
  return (
    <ScrollView style={styles.container} refreshControl={
      // 这里可以实现下拉刷新功能
      <RefreshControl
        refreshing={data.networkStatus === 4}
        onRefresh={data.refetch}
      />
    }>
      <Text style={styles.title}>GitHunt</Text>
      <FeedList data={data} />
      <Text style={styles.fullApp}>See the full app at www.githunt.com</Text>
      <Button
        buttonStyle={styles.learnMore}
        onPress={goToApolloWebsite}
        icon={{name: 'code'}}
        raised
        backgroundColor="#22A699"
        title="Learn more about Apollo"
      />
    </ScrollView>
  );
}
```

最后，我们来看实际显示项目的 `FeedList` 组件。数据来源于 Apollo 的 `data` prop，并将数据映射到了显示列表项。你可以看到，多亏了 GraphQL，我们得到的数据形状完全符合我们预期。

```js
function FeedList({ data }) {
  if (data.networkStatus === 1) {
    return <ActivityIndicator style={styles.loading} />;
  }

  if (data.error) {
    return <Text>Error! {data.error.message}</Text>;
  }

  return (
    <List containerStyle={styles.list}>
      { data.feed.map((item) => {
          const badge = item.repository.stargazers_count && {
            value: `☆ ${item.repository.stargazers_count}`,
            badgeContainerStyle: { right: 10, backgroundColor: '#56579B' },
            badgeTextStyle: { fontSize: 12 },
          };

          return <ListItem
            hideChevron
            title={`${item.repository.owner.login}/${item.repository.name}`}
            subtitle={`Posted by ${item.postedBy.login}`}
            badge={badge}
          />;
        }
      ) }
    </List>
  )
}
```

现在，你已经浏览了编写 React Native 应用所需的所有代码，该应用加载一个项目列表，并将其显示在具有下拉刷新功能的列表中。正如你所看到的，我们只需写很少的数据加载代码！我们认为 Apollo 和 GraphQL 可以帮助你从数据加载的方式中解脱，方便你更快地编写应用。

<h2 id="next-steps">下一步</h2>

让我们从头开始构建你自己的应用！这有两个教程可供你学习，我们建议按以下顺序进行：

1. [全栈 GraphQL + React 教程](https://dev-blog.apollodata.com/full-stack-react-graphql-tutorial-582ac8d24e3b#.cwvxzphyc)，出自 [Jonas Helfer](https://twitter.com/helferjs)，Apollo 客户端的首席开发人员。
2. 由 [Graphcool](https://www.graph.cool/) 团队和社区开发的 [How to GraphQL](https://www.howtographql.com/react-apollo/0-introduction/), Graphcool 是一个后端托管平台。

玩的开心！