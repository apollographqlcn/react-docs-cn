---
title: 在 TypeScript 和 Flow 中使用 Apollo
sidebar_title: 使用 TypeScript 和 Flow
---

随着应用程序规模的增长，您可能会需要引入一个类型系统来提升开发效率。Apollo 同时支持 Flow 和 TypeScript 两种类型定义系统。 `apollo-client` 和 `react-apollo` 都在它们的 npm 包中包含了类型定义，所以在你的项目中引入这些库后，类型定义文件也都安装好了。

本篇文档假设您已经在项目中配置好了 Flow 或 TypeScript，如果没有的话请查看 [Flow](https://flow.org/en/docs/install/) 或 [TypeScript](https://github.com/Microsoft/TypeScript-React-Conversion-Guide#typescript-react-conversion-guide) 的配置文档。

<h2 id="operation-result">操作结果</h2>

使用 GraphQL 类型系统时最常见的需求是标注操作结果的类型。鉴于 GraphQL 服务端的 schema 是强类型的，我们甚至可以用 [apollo-codegen](https://github.com/apollographql/apollo-codegen)  等工具自动生成 Flow 或 TypeScript 类型定义。然而，在本篇文档中，我们将手动编写结果类型。

由于查询的结果将被发送到包装组件作为 props，我们希望能够告诉我们的类型系统这些 props 的结构。以下是使用 Flow 设置类型的操作示例：

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

The same example looks like this when using TypeScript:
使用TypeScript时，同样的例子如下所示：

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

One of the major differences between the two systems is how they handle inferring types. Because TypeScript does not infer types, the React integration of Apollo exports extra type definitions to make adding types easier.
两个系统之间的主要区别之一是它们如何处理推断类型。 因为TypeScript不推断类型，所以Apollo的React集成导出额外的类型定义，使添加类型更容易。

<h2 id="options">选项</h2>

Typically, variables to the query will be computed from the props of the wrapper component. Wherever the component is used in your application, the caller would pass arguments that we want our type system to validate what the shape of these props could look like. Here is an example setting the type of props using Flow:
通常，查询的变量将根据包装器组件的属性计算。以下是使用Flow设置道具类型的示例：

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

The same example looks like this when using TypeScript:
使用TypeScript时，同样的例子如下所示：

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
当访问通过道具传递给组件的深层嵌套对象时，这是非常有用的。 例如，当添加道具类型时，使用Flow的项目将开始显示错误，其中传递的道具是无效的：

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

One of the most powerful feature of the React integration is the `props` function which allows you to reshape the result data from an operation into a new shape of props for the wrapped component. GraphQL is awesome at allowing you to only request the data you want from the server. The client still often needs to reshape or do client side calculations based on these results. The return value can even differ depending on the state of the operation (i.e loading, error, recieved data), so informing our type system of choice of these possible values is really important to make sure our components won't have runtime errors.
React集成最强大的功能之一是“props”功能，可以让您将操作中的结果数据重新形成包装组件的新形状的道具。 GraphQL非常棒，允许您只从服务器请求您想要的数据。 客户端仍然需要根据这些结果重塑或进行客户端计算。 返回值甚至可以根据操作的状态（即加载，错误，接收的数据）而有所不同，因此通知我们的类型系统选择这些可能的值非常重要，以确保我们的组件不会有运行时错误。

The `graphql` wrapper from `react-apollo` supports manually declaring the shape of your result props. It is implmented in Flow like this:
“react-apollo”中的“graphql”包装器支持手动声明结果道具的形状。 在流中像这样：

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

The same example looks like this when using TypeScript:
使用TypeScript时，同样的例子如下所示：

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

Since we have typed the response shape, the props shape, and the shape of what will be passed to the client, we can prevent errors in multiple places. Our options and props function within the `graphql` wrapper are now type safe, our rendered component is protected, and our tree of components have their required props enforced. Take for example errors using Flow within the `props` function after adding the above typings:
由于我们输入了响应形状，道具形状以及将传递给客户端的形状，我们可以防止多个地方的错误。 我们在“graphql”包装器中的选项和道具功能现在是类型安全的，我们渲染的组件受到保护，我们的组件树已经执行了他们所需的道具。 在添加以上打字之后，使用“props”功能中的Flow作为示例：

```javascript
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

<h2 id="classes-vs-functions">classes vs functions</h2>

All of the above examples show wrapping a component which is just a function using the result of a `graphql` wrapper. Sometimes, components that depend on GraphQL data require state and are formed using the `class MyComponent extends React.Component` practice. In these use cases, both TypeScript and Flow require adding prop shape to the class instance. In order to support this, `react-apollo` exports types to support creating result types easily. This is the previous example shortened to show just the component when using Flow:
以上所有示例都显示了使用“graphql”包装器的结果仅仅是一个函数的组件。 有时，依赖于GraphQL数据的组件需要状态，并使用“MyComponent extends React.Component”实践形成。 在这些用例中，TypeScript和Flow都需要向类实例添加支持形状。 为了支持这一点，“react-apollo”导出类型可以轻松地创建结果类型。 这是前面的例子，简化为在使用Flow时只显示组件：

```javascript
import { ChildProps } from "react-apollo";

const withCharacter: OperationComponent<Response, InputProps> = graphql(HERO_QUERY, {
  options: ({ episode }) => ({
    variables: { episode }
  })
});

export default class Character extends Component {
    props: ChildProps<InputProps, Response>;

    render(){
        const { loading, hero, error } = this.props.data;
        if (loading) return <div>Loading</div>;
        if (error) return <h1>ERROR</h1>;
        return ...// 具有数据的实际组件;
    }
}
```

The same example looks like this when using TypeScript:
使用TypeScript时，同样的例子如下所示：

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
If you are using the `name` property in the configuration of the `graphql` wrapper, you will need to manually attach the type of the response to the `props` function. An example using TypeScript would be like this:
如果在“graphql”包装器的配置中使用`name`属性，则需要手动将响应类型附加到`props`函数。 使用TypeScript的例子就是这样的：

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

<h2 id="more-info">更多信息</h2>

For more information regarding using Flow or TypeScript with Apollo, check out the following articles:
有关在Apollo中使用Flow或TypeScript的更多信息，请查看以下文章：
- [一个更强大的（强类型）React Apollo](https://dev-blog.apollodata.com/a-stronger-typed-react-apollo-c43bd52be0d8)
- [TypeScript 和 React Apollo 快速上手](https://dev-blog.apollodata.com/getting-started-with-typescript-and-apollo-a9aa2c7dcf87)


