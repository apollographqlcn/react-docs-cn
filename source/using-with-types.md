---
title: 在 TypeScript 和 Flow 中使用 Apollo
sidebar_title: 使用 TypeScript 和 Flow
---

随着应用程序规模的增长，你可能会需要引入一个类型系统来提升开发效率。Apollo 同时支持 Flow 和 TypeScript 两种类型定义系统。 `apollo-client` 和 `react-apollo` 都在它们的 npm 包中包含了类型定义，所以在你的项目中引入这些库后，类型定义文件也都安装好了。

本篇文档假设你已经在项目中配置好了 Flow 或 TypeScript，如果没有的话请查看 [Flow](https://flow.org/en/docs/install/) 或 [TypeScript](https://github.com/Microsoft/TypeScript-React-Conversion-Guide#typescript-react-conversion-guide) 的配置文档。

<h2 id="operation-result">操作结果</h2>

使用 GraphQL 类型系统时最常见的需求是标注操作结果的类型。鉴于 GraphQL 服务端的 schema 是强类型的，我们甚至可以用 [apollo-codegen](https://github.com/apollographql/apollo-codegen)  等工具自动生成 Flow 或 TypeScript 类型定义。然而，在本篇文档中，我们将手动编写结果类型。

由于查询的结果将被发送到包裹组件作为 props，我们希望能够告诉我们的类型系统这些 props 的模型。以下是使用 Flow 设置类型的操作示例：

```javascript
// @flow
import React from "react";
import gql from "graphql-tag";
import { graphql } from "react-apollo";

import type { OperationComponent } from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string,
  id: string,
  appearsIn: string[],
  friends: Hero[]
};

type Response = {
  hero: Hero
};

const withCharacter: OperationComponent<Response> = graphql(HERO_QUERY, {
  options: () => ({
    variables: { episode: "JEDI" }
  })
});

export default withCharacter(({ data: { loading, hero, error } }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// 具有数据的实际组件;
});
```

使用 TypeScript 时，同样的例子如下所示：

```javascript
import React from "react";
import gql from "graphql-tag";
import { graphql } from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string;
  id: string;
  appearsIn: string[];
  friends: Hero[];
};

type Response = {
  hero: Hero;
};

const withCharacter = graphql<Response>(HERO_QUERY, {
  options: () => ({
    variables: { episode: "JEDI" }
  })
});

export default withCharacter(({ data: { loading, hero, error } }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// 具有数据的实际组件;
});
```

两个类型系统之间的主要区别之一是它们如何处理推断类型。 因为 TypeScript 不做类型推断，所以集成了 Apollo 的 React 代码导出额外的类型定义，使得添加类型更容易。

<h2 id="options">选项</h2>

通常，查询的变量将由包裹组件的 props 计算而来。以下是使用 Flow 设置 props 类型的示例：

```javascript
// @flow
import React from "react";
import gql from "graphql-tag";
import { graphql } from "react-apollo";

import type { OperationComponent } from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string,
  id: string,
  appearsIn: string[],
  friends: Hero[]
};

type Response = {
  hero: Hero
};

export type InputProps = {
  episode: string
};

const withCharacter: OperationComponent<Response, InputProps> = graphql(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  })
});

export default withCharacter(({ data: { loading, hero, error } }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// 具有数据的实际组件;
});
```

使用 TypeScript 时，同样的例子如下所示：

```javascript
import React from "react";
import gql from "graphql-tag";
import { graphql } from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string;
  id: string;
  appearsIn: string[];
  friends: Hero[];
};

type Response = {
  hero: Hero;
};

type InputProps = {
  episode: string
};

const withCharacter = graphql<Response, InputProps>(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  }),
});

export default withCharacter(({ data: { loading, hero, error } }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// 具有数据的实际组件;
});
```

This is expecially helpful when accessing deeply nested objects that are passed down to the component through props. For example, when adding prop types a project using Flow will begin to surface errors where props being passed are invalid:
当访问通过 props 传递给组件的深层嵌套对象时，适用 TypeScript 非常有用的。 例如，当添加 prop 类型时，使用 Flow 的项目将开始报错，显示传递的 props 是无效的：

```javascript
// @flow
import React from "react";
import ApolloClient, { createNetworkInterface } from "apollo-client";
import { ApolloProvider } from "react-apollo";

import Character from "./Character";

export const networkInterface = createNetworkInterface({
  uri: "https://mpjk0plp9.lp.gql.zone/graphql"
});

export const client = new ApolloClient({ networkInterface });

export default () =>
  <ApolloProvider client={client}>
    // $ExpectError property `episode`. Property not found in. See: src/Character.js:43
    <Character />
  </ApolloProvider>;
```

<h2 id="props">props</h2>

React 与 Apollo 结合的应用最强大的功能之一，便是 `props` 允许你将操作中的结果数据重新组装成包裹组件所需模型的新的 props。 GraphQL 在允许你从服务端请求你想要的数据方面非常棒，然而客户端仍然需要根据这些结果调整数据模型或进行计算。请求的返回值甚至可以根据操作的状态（即加载中，错误，接收的数据）而有所不同，因此在类型声明中标识这些可能的值非常重要，以确保我们的组件不会有运行时错误。

`react-apollo` 中的 `graphql` 包裹器支持手动声明结果 props 的模型。在 Flow 中像这样：

```javascript
// @flow
import React from "react";
import gql from "graphql-tag";
import { graphql } from "react-apollo";

import type { OperationComponent, QueryProps } from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string,
  id: string,
  appearsIn: string[],
  friends: Hero[]
};

type Response = {
  hero: Hero
};

type Props = Response & QueryProps;

export type InputProps = {
  episode: string
};

const withCharacter: OperationComponent<Response, InputProps, Props> = graphql(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  }),
  props: ({ data }) => ({ ...data })
});

export default withCharacter(({ loading, hero, error }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// 具有数据的实际组件;
});
```

使用 TypeScript 时，同样的例子如下所示：

```javascript
import React from "react";
import gql from "graphql-tag";
import { graphql, NamedProps, QueryProps} from "react-apollo";

const HERO_QUERY = gql`
  query GetCharacter($episode: Episode!) {
    hero(episode: $episode) {
      name
      id
      friends {
        name
        id
        appearsIn
      }
    }
  }
`;

type Hero = {
  name: string;
  id: string;
  appearsIn: string[];
  friends: Hero[];
};

type Response = {
  hero: Hero;
};

type WrappedProps = Response & QueryProps;

type InputProps = {
  episode: string
};

const withCharacter = graphql<Response, InputProps, WrappedProps>(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  }),
  props: ({ data }) => ({ ...data })
});

export default withCharacter(({ loading, hero, error }) => {
  if (loading) return <div>Loading</div>;
  if (error) return <h1>ERROR</h1>;
  return ...// 具有数据的实际组件;
});
```

由于声明了响应模型，props 模型以及将传递给客户端的模型的类型，我们可以防止多处的错误。我们在 `graphql` 包裹器中的选项和 props 函数现在是类型安全的，我们渲染的组件受到保护，我们的组件树已经强制声明了它们所需的 props。 在上述使用 Flow 的示例中，添加如下代码到 `props` 函数展示错误信息：

```javascript
// @flow

export const withCharacter: OperationComponent<Response, InputProps, Props> = graphql(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  }),
  props: ({ data, ownProps }) => ({
    ...data,
    // $ExpectError [string] This type cannot be compared to number
    episode: ownProps.episode > 1,
    // $ExpectError property `isHero`. Property not found on object type
    isHero: data && data.hero && data.hero.isHero
  })
});
```

With this addition, the entirety of the integration between Apollo and React can be statically typed. When combined with the strong tooling each system provides, it can make for a much improved application and developer experience.
通过这一补充，可以静态地输入Apollo和React之间的整体整合。 当与每个系统提供的强大的工具相结合时，它可以改善应用程序和开发人员的体验。

<h2 id="classes-vs-functions">classes 对比 functions</h2>

All of the above examples show wrapping a component which is just a function using the result of a `graphql` wrapper. Sometimes, components that depend on GraphQL data require state and are formed using the `class MyComponent extends React.Component` practice. In these use cases, both TypeScript and Flow require adding prop shape to the class instance. In order to support this, `react-apollo` exports types to support creating result types easily. This is the previous example shortened to show just the component when using Flow:
以上所有示例都是使用 `graphql` 高阶组件包裹函数组件。有时，依赖于 GraphQL 数据的组件需要 state，并使用 `class MyComponent extends React.Component` 这样的方式声明。在这种场景下，TypeScript 和 Flow 都需要向类实例添加 prop 模型。为了实现这一点，`react-apollo` 导出的类型可以轻松创建结果类型。简化之前的例子只显示组件，如下是使用 Flow 时的情况：

```javascript
// @flow

import { ChildProps } from "react-apollo";

const withCharacter: OperationComponent<Response, InputProps> = graphql(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  })
});

// flow will infer this type 
export default class Character extends Component {
  render(){
    const { loading, hero, error } = this.props.data;
    if (loading) return <div>Loading</div>;
    if (error) return <h1>ERROR</h1>;
    return ...// 具有数据的实际组件;
  }
}
 
const CharacterWithData = withCharacter(Character);
```

使用 TypeScript 时，同样的例子如下所示：

```javascript
import { ChildProps } from "react-apollo";

const withCharacter = graphql<Response, InputProps>(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  })
});

export default class Character extends React.Component<ChildProps<InputProps, Response>, {}> {
  render(){
    const { loading, hero, error } = this.props.data;
    if (loading) return <div>Loading</div>;
    if (error) return <h1>ERROR</h1>;
    return ...// 具有数据的实际组件;
  }
}
```

<h2 id="using-name">使用 `name` 属性</h2>

如果在 `graphql` 包裹器的配置中使用 `name` 属性，则需要手动将响应类型附加到 `props` 函数。 使用 TypeScript 时示例如下：

```javascript
import { NamedProps, QueryProps } from 'react-apollo';

export const withCharacter = graphql<Response, InputProps, Prop>(HERO_QUERY, {
  name: 'character',
  props: ({ character, ownProps }: NamedProps<{ character: QueryProps & Response }, Props) => ({
    ...character,
    // $ExpectError [string] This type cannot be compared to number
    episode: ownProps.episode > 1,
    // $ExpectError property `isHero`. Property not found on object type
    isHero: character && character.hero && character.hero.isHero
  })
});
```

<h2 id="more-info">扩展阅读</h2>

有关在 Apollo 中使用 Flow 或 TypeScript 的更多信息，请查看以下文章：
- [一个更强大的（强类型）React Apollo](https://dev-blog.apollodata.com/a-stronger-typed-react-apollo-c43bd52be0d8)
- [TypeScript 和 React Apollo 快速上手](https://dev-blog.apollodata.com/getting-started-with-typescript-and-apollo-a9aa2c7dcf87)


