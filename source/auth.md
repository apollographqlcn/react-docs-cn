---
title: 认证
---

除非您正在加载的所有数据都是完全公开的，否则您的应用程序有某种用户，帐户和权限系统。如果不同的用户在应用程序中具有不同的权限，那么您需要一种告诉服务器哪个用户与每个请求相关联的方法。

Apollo Client配有可插拔的[HTTP网络接口](/core/network.html)，其中包含多个验证选项。

## Cookie

如果您的应用程序是基于浏览器的，并且您正在使用Cookie进行登录和会话管理（后端），则可以轻松地将网络界面与每个请求一起发送。你只需要传递凭据选项。例如`{ credentials: 'same-origin' }`如下所示，如果您的后端服务器是相同的域，或者如果您的后端是不同的域，则`{ credentials: 'include' }`。

```js
const networkInterface = createNetworkInterface({
  uri: '/graphql',
  opts: {
    credentials: 'same-origin',
  },
});

const client = new ApolloClient({
  networkInterface,
});
```

该选项只需传递到发送查询时由网络接口​​使用的[`fetch` polyfill](https://github.com/github/fetch)。

注意：后端还必须允许来自请求的来源的凭据。例如如果在node.js中使用npm中流行的 'cors' 软件包，以下设置将与上述apollo客户端设置配合使用，

```js
// 启用cors
var corsOptions = {
  origin: '<insert uri of front-end domain>',
  credentials: true // <-- 所需的后端设置
};
app.use(cors(corsOptions));
```

## Header

在使用HTTP时识别自己的另一种常见方法是沿着授权头发送。 Apollo网络接口具有中间件功能，可让您在发送到服务器之前修改请求。为每个HTTP请求添加一个`authorization`标签是很容易的。在这个例子中，每次发送请求时，我们都会从`localStorage`中拉取登录凭证：

```js
import { ApolloClient, createNetworkInterface } from 'react-apollo';

const networkInterface = createNetworkInterface({
  uri: '/graphql',
});

networkInterface.use([{
  applyMiddleware(req, next) {
    if (!req.options.headers) {
      req.options.headers = {};  // 如果需要，创建 header 对象。
    }

    // 从本地存储获取认证令牌（如果存在）
    const token = localStorage.getItem('token');
    req.options.headers.authorization = token ? `Bearer ${token}` : null;
    next();
  }
}]);

const client = new ApolloClient({
  networkInterface,
});
```

服务器可以使用该标头来验证用户并将其附加到GraphQL执行上下文，因此解析器可以根据用户的角色和权限修改其行为。

<h2 id="login-logout">登出时重置 Store</h2>

由于Apollo缓存了所有查询结果，所以当登录状态发生变化时，重要的是删除它们。

确保UI和存储状态反映当前用户权限的最简单方法是在登录或注销过程完成后调用 `client.resetStore()`。这将导致存储被清除，并且所有活动查询被重新获取。该组件必须包含在`withApollo` 高阶组件中，以通过道具直接访问 `Apolloclient`。

另一个选择是重新加载页面，这将具有类似的效果。

```js
import { withApollo, graphql } from 'react-apollo';
import ApolloClient from 'apollo-client';

class Profile extends React.Component {
  constructor(props) {
    super(props);

    this.logout = () => {
      App.logout() // 或其他任何注销流程
      .then(() =>
        props.client.resetStore();
      )
      .catch(err =>
        console.error('Logout failed', err);
      );
    }
  }

  render() {
    const { loading, currentUser } = this.props;

    if (loading) {
      return (
        <p className="navbar-text navbar-right">
          Loading...
        </p>
      );
    } else if (currentUser) {
      return (
        <span>
          <p className="navbar-text navbar-right">
            {currentUser.login}
            &nbsp;
            <button onClick={this.logout}>Log out</button>
          </p>
        </span>
      );
    }
    return (
      <p className="navbar-text navbar-right">
        <a href="/login/github">Log in with GitHub</a>
      </p>
    );
  }
}

Profile.propTypes = {
  client: React.PropTypes.instanceOf(ApolloClient),
  loading: React.PropTypes.bool,
  currentUser: React.PropTypes.shape({
    login: React.PropTypes.string.isRequired,
  }),
};

const PROFILE_QUERY = gql`
  query CurrentUserForLayout {
    currentUser {
      login
      avatar_url
    }
  }
`;

export default withApollo(graphql(PROFILE_QUERY, {
  options: { forceFetch: true },
  props: ({ data: { loading, currentUser } }) => ({
    loading, currentUser,
  }),
})(Profile));
```
