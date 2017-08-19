---
title: 安装和配置
---
<h2 id="installation">安装</h2>

要开始使用 Appollo 和 React，请安装 `react-apollo` npm 包。虽然后面还会安装几个包，不过这个 npm 包足够你起步使用了。

```bash
npm install react-apollo --save
```

> 注意：想在 React Native 中使用 Apollo Client，不需要做额外的操作，只需像往常一样安装和导入。

要开始在 React 中使用 Apollo，我们需要创建一个 `ApolloClient` 和 `ApolloProvider`

- `ApolloClient` 作为查询结果数据的中心存储，缓存并分发查询结果。
- `ApolloProvider` 使该客户端实例可用于多层的 React 组件中。

<h2 id="creation-client">创建客户端</h2>

先创建一个[`ApolloClient`](/core/apollo-client-api.html＃constructor)实例，并将其指向你的 GraphQL 服务器：

```js
import { ApolloClient } from 'react-apollo';

// 默认情况下，该客户端将发送查询同个域名下的
// `/ graphql` 路径
const client = new ApolloClient();
```

客户端支持各种[配置](/core/apollo-client-api.html＃构造函数)，特别要注意的是，如果要更改 GraphQL 服务器的 URL，你可以创建一个自定义的[`NetworkInterface](/core/apollo-client-api.html#NetworkInterface)：

```js
import { ApolloClient, createNetworkInterface } from 'react-apollo';

const networkInterface = createNetworkInterface({
  uri: 'http://api.example.com/graphql'
});

const client = new ApolloClient({
  networkInterface: networkInterface
});
```

`ApolloClient`还有一些控制客户端行为的选项，我们将在本指南中展示一些示例。

<h2 id="creation-provider">创建提供者</h2>

要将客户端实例连接到组件树，请使用 `ApolloProvider` 组件。我们建议将 `ApolloProvider` 放置组件层级的高阶部分，也就是放在需要访问 GraphQL 数据的组件之上。例如，如果你使用 React Router，它可能就放在根路由组件之上。

```js
import { ApolloClient, ApolloProvider } from 'react-apollo';

// 如上所述创建客户端 
const client = new ApolloClient();

ReactDOM.render(
  <ApolloProvider client={client}>
    <MyAppComponent />
  </ApolloProvider>,
  document.getElementById('root')
)
```

<h2 id="connections-data">请求数据</h2>

推荐使用 `graphql()` 容器的方式来获取数据或执行 muitation。这是一个React [高阶组件](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html#subscriptions-and-side-effects)，并与被包裹的组件通过 props 通信。

`graphql()` 的基本用法如下：

```js
import React, { Component } from 'react';
import { gql, graphql } from 'react-apollo';

// MyComponent 是一个纯 UI 组件，跟 Apollo 无关
const MyComponent = (props) => (
  <div>...</div>
);

// 使用 `gql` 标签初始化GraphQL查询或 mutation
const MyQuery = gql`query { todos { text } }`;
const MyMutation = gql`mutation { addTodo(text: "Test 123") { id } }`;

// 然后我们可以使用 `graphql` 将 MyQuery 返回的查询结果 通过 props给// 传递给 MyComponent（随着数据变化而更新）
const MyComponentWithData = graphql(MyQuery)(MyComponent);

// 或者，我们可以将 MyMutation 的执行绑定到一个 prop
const MyComponentWithMutation = graphql(MyMutation)(MyComponent);
```

如果你在 React 组件类使用[ES2016 装饰器](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841#.nn723s5u2)，你应该会更喜欢装饰器这种方便的语法，就像下面代码一样：

```js
import React, { Component } from 'react';
import { graphql } from 'react-apollo';

@graphql(MyQuery)
@graphql(MyMutation)
class MyComponent extends Component {
  render() {
    return <div>...</div>;
  }
}
```

在本指南中，我们不会使用装饰器语法来简化代码，不过你可以把例子的代码改用装饰器的写法。


<h2 id="fragment-matcher">在 union 和 interface 上使用 Fragment </h2>

默认情况下，Apollo Client 不需要任何有关 GraphQL 架构的知识，这意味着非常容易搭建，能在各种服务器下使用，无论你的 shcema 有多复杂都能支持。然而，随着对 Apollo 和 GraphQL 越来越熟练，你可能会开始在 union 或 interface 上使用 Fragment。以下是在 interface 上使用 Fragment 的查询示例：

```
query {
  all_people {
    ... on Character {
      name
    }
    ... on Jedi {
      side
    }
    ... on Droid {
      model
    }
  }
}
          
```

在上面的查询中，`all_people` 返回类型为 `Character []` 的结果。 `Jedi` 和 `Droid` 可以是 `Character` 的具体类型，但是在客户端上没有办法知道关于 schema 的信息。默认情况下，Apollo Client 将使用启发式 fragment 匹配器，如果返回的结果包含了 fragment 的字段集合则表示匹配，如果返回数据少了一些字段，则表示不匹配。这在大多数情况下是靠谱的，但这也意味着Apollo Client 无法检查你的服务器返回结果，当你使用 `update`，`updateQuery`，`writeQuery` 手动将无效数据写入 store 时也没法告诉你。

以下部分说明如何将必要的 schema 信息告诉 Apollo Client，然后 union 和 interface 就可以准确匹配，在数据写入 store 前对结果校验。

为了支持结果校验和在 union 和 interfaces 上准确地匹配 fragment，可以使用一个名为 `IntrospectionFragmentMatcher` 的特殊 fragment 匹配器。请按照以下三个步骤来使用：

1. 查询你的 服务器/schema，获取关于 union 和 interface 的必要信息：

```graphql
{
  __schema {
    types {
      kind
      name
      possibleTypes {
        name
      }
    }
  }
}
```

2.使用刚获得的信息创建一个新的 IntrospectionFragment 匹配器（你可以根据需要过滤 INTERFACE 或 UNION 类型）：


```js
import { IntrospectionFragmentMatcher } from 'react-apollo';

const myFragmentMatcher = new IntrospectionFragmentMatcher({
  introspectionQueryResultData: {
    __schema: {
      types: [
        {
          kind: "INTERFACE",
          name: "Character",
          possibleTypes: [
            { name: "Jedi" },
            { name: "Droid" },
          ],
        }, // 这只是一个例子，把你自己的 INTERFACE 和 UNION 类型放在这里！
      ],
    },
  }
})
```

3.  将新建的 `IntrospectionFragmentMatcher` 传递给 Apollo Client 构造器：

```js
const client = new ApolloClient({
  fragmentMatcher: myFragmentMatcher,
});
```

如果 schema 更新了 union 或 interface 类型，则必须相应地更新 fragment 匹配器。为了保证信息的自动更新，我们建议你设置构建程序，从 schema 中提取必要的信息，并将其作为 JSON 文件包含在客户端软件包中，在构建 fragment 匹配器时可以传入。

（注意：如果有人已经搭建了一些构建程序，请考虑在这份文档发起 PR，与社区伙伴分享你的经验！）
