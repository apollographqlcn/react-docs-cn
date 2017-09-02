---
title: 认证
---

除非你正在加载的所有数据都是完全公开的，否则你的应用应该有某种用户，帐户和权限系统。如果不同的用户在应用中具有不同的权限，那么你需要一种让服务器知道每个请求与哪个用户相关联的方法。

Apollo 客户端配有可插拔的 [HTTP 网络接口](http://dev.apollodata.com/core/network.html)，其中包含多个验证选项。

## Cookie

如果你的应用是基于浏览器的，并且你正在使用 cookie 与后端进行登录和会话管理，则很容易让网络接口将 cookie 与请求一起发送。你只需要传递凭据选项。例如下面所示的，如果你的后端服务器是在同一个域，那么设置`{ credentials: 'same-origin' }`，或者如果你的后端是不同的域，则设置 `{ credentials: 'include' }`。

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

该选项只是简单地传递给发送查询时由网络接口​​使用的[`fetch` polyfill](https://github.com/github/fetch)。

注意：后端还必须允许来自请求来源的凭据。例如，如果在 Node.js 中使用 NPM 中流行的 'cors' 软件包，以下设置将与上述 Apollo 客户端设置配合使用。

```js
// 启用 cors
var corsOptions = {
  origin: '<insert uri of front-end domain>',
  credentials: true // <-- 所需的后端设置
};
app.use(cors(corsOptions));
```

## Header

另一种使用 HTTP 认证的常见方法是发送一个授权请求头。Apollo 网络接口具有中间件功能，可让你将请求发送到服务器之前修改它。因此，为每个 HTTP 请求添加一个 `authorization` 请求头是很容易的。在下面这个例子中，每次发送请求时，我们都会从 `localStorage` 中获取登录令牌：

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

    // 从 localStorage 获取认证令牌（如果存在）
    const token = localStorage.getItem('token');
    req.options.headers.authorization = token ? `Bearer ${token}` : null;
    next();
  }
}]);

const client = new ApolloClient({
  networkInterface,
});
```

服务器可以使用该请求头来授权用户并将其附加到 GraphQL 执行上下文，因此解析器可以根据用户的角色和权限修改其行为。

<h2 id="login-logout">登出时重置 store</h2>

由于 Apollo 缓存了所有查询结果，所以当登录状态发生变化时，首先要删除它们。

确保 UI 和 store 状态与当前用户权限一致的最简单方法，是在登录或注销过程完成后调用 `client.resetStore()` 方法。这将导致 store 被清空，并且所有激活的查询将被重新获取。故组件必须包含在 `withApollo` 高阶组件中，以通过 props 直接访问 `Apolloclient`。

另一个选择是重新加载页面，也能达到类似的效果。

```js
import { withApollo, graphql, gql } from 'react-apollo';
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
