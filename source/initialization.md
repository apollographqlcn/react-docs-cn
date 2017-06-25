---
title: 设置和选项
---
<h2 id="installation">安装</h2>

要开始使用阿波罗和React，请安装“react-apollo”npm软件包。这会导出您需要开始的所有内容，即使在引擎盖下有几个包。

```bash
npm install react-apollo --save
```

> 注意：您无需做任何特别的操作，让Apollo Client在React Native中工作，只需像往常一样安装和导入。

要开始使用Apollo with React，我们需要创建一个“ApolloClient”和“ApolloProvider”。

- “ApolloClient”作为查询结果数据的中央存储库，缓存并分发查询结果。
- “ApolloProvider”使该客户端实例可用于我们的React组件层次结构。

<h2 id="creation-client">创建客户端</h2>

要开始创建一个[`ApolloClient`](/core/apollo-client-api.html＃constructor)实例，并将其指向您的GraphQL服务器：

```js
import { ApolloClient } from 'react-apollo';

// 默认情况下，该客户端将发送查询
// `/ graphql`端点在同一台主机上
const client = new ApolloClient();
```

客户端使用各种[options](/core/apollo-client-api.html＃构造函数)，但特别是如果要更改GraphQL服务器的URL，可以创建一个自定义的[`NetworkInterface](/core/apollo-client-api.html#NetworkInterface)：

```js
import { ApolloClient, createNetworkInterface } from 'react-apollo';

const networkInterface = createNetworkInterface({
  uri: 'http://api.example.com/graphql'
});

const client = new ApolloClient({
  networkInterface: networkInterface
});
```

`ApolloClient`还有一些控制客户端行为的选项，我们将在本指南中看到他们使用的例子。

<h2 id="creation-provider">创建提供者</h2>

要将客户端实例连接到组件树，请使用“ApolloProvider”组件。我们建议将“ApolloProvider”放置在您的视图层次结构中的某处，您需要访问GraphQL数据的任何地方。例如，如果您使用React Router，它可能位于根路由组件之外。

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

“graphql()”容器是推荐的方法来获取数据或进行突变。这是一个React [高级组件](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html#subscriptions-and-side-effects)，并与包裹的组件通过道具。

`graphql()`的基本用法如下：

```js
import React, { Component } from 'react';
import { gql, graphql } from 'react-apollo';

// MyComponent是一个演示组件，不知道 Apollo
const MyComponent = (props) => (
  <div>...</div>
);

// 使用`gql`标签初始化GraphQL查询或突变
const MyQuery = gql`query { todos { text } }`;
const MyMutation = gql`mutation { addTodo(text: "Test 123") { id } }`;

// 然后我们可以使用`graphql`将MyQuery返回的查询结果传递给MyComponent作为支持（并将其更新为结果更改）
const MyComponentWithData = graphql(MyQuery)(MyComponent);

// 或者，我们可以将MyMutation的执行绑定到一个prop
const MyComponentWithMutation = graphql(MyMutation)(MyComponent);
```

如果您使用React类组件使用[ES2016装饰器](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841#.nn723s5u2)，则可能更喜欢使用更简洁的装饰器语法一样：

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

在本指南中，我们不会使用装饰器语法来使代码更加平易近人，但是可以将任何示例转换为使用装饰器。


<h2 id="fragment-matcher">在工会和接口上使用Fragments </h2>

默认情况下，Apollo Client不需要任何有关GraphQL架构的知识，这意味着设置和使用任何服务器非常容易，甚至支持最大的模式。然而，由于您对Apollo和GraphQL的使用变得更加复杂，您可能会开始在界面或工会上使用片段。以下是在界面上使用片段的查询示例：

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

在上面的查询中，`all_people`返回类型为`Character []`的结果。 “Jedi”和“Droid”可以是“Character”的具体类型，但是在客户端上没有办法知道没有关于模式的一些信息。默认情况下，Apollo Client将使用启发式片段匹配器，如果结果包含其选择集中的所有字段，则假定片段匹配，并且在任何字段丢失时不匹配。这在大多数情况下起作用，但这也意味着Apollo Client无法检查您的服务器响应，并且无法使用`update`，`updateQuery`，`writeQuery`手动将无效数据写入商店，等等

以下部分说明如何将必要的架构知识传递给Apollo Client，因此可以将工会和接口在将其写入商店之前进行准确匹配和结果验证。

为了支持结合验证和在工会和接口上的准确的片段匹配，可以使用一个名为“IntrospectionFragmentMatcher”的特殊片段匹配器。要设置，请按照以下三个步骤：

1.查询您的服务器/架构以获取关于联合和接口的必要信息：

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

2.使用刚获得的信息创建一个新的IntrospectionFragment匹配器（您可以根据需要过滤任何不属于INTERFACE或UNION的类型）：


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
        }, // //这只是一个例子，把你自己的INTERFACE和UNION类型放在这里！
      ],
    },
  }
})
```

3. 在建造期间将新建的“IntrospectionFragmentMatcher”传递给Apollo Client：

```js
const client = new ApolloClient({
  fragmentMatcher: myFragmentMatcher,
});
```

如果在模式中有任何与联合或接口类型相关的更改，则必须相应地更新片段匹配器。为了保持此信息的自动更新，我们建议您设置构建步骤，从模式中提取必要的信息，并将其作为JSON文件包含在客户端软件包中，并从构建片段匹配器时导入。

（注意：如果任何人已经建立了一个建立步骤，请考虑在这里的文档制作公关，与社区的其他人分享您的说明！）
