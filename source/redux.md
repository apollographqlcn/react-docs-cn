---
title: 与 Redux 集成
---

默认情况下，Apollo Client创建自己的内部Redux store 来管理查询及其结果。如果您已经在应用程序的其余部分使用Redux，则可以将客户端与现有存储集成。

> 注意：虽然这将使Apollo Client将其数据保存在同一个存储中，但您仍然应该使用[graphql container](/react/higher-order-components.html) 将数据附加到UI中。如果要在组件中使用Redux和Apollo状态，则需要_同时_使用`graphql`的react-apollo 和 react-redux 的`connect`。

这将让您更好地跟踪应用程序中发生的不同事件，以及客户端和服务器端数据如何更改交错。它也将使用像[Redux 开发者工具](https://github.com/zalmoxisus/redux-devtools-extension)这样的工具更自然。

<h2 id="creating-a-store">创建一个 Store</h2>

如果您想使用自己的 Store，您需要从Apollo Client实例中传递 reducer 和中间件；您可以直接将 Store 传递到您的 `ApolloProvider` 中：

```js
import { createStore, combineReducers, applyMiddleware, compose } from 'redux';
import { ApolloClient, ApolloProvider } from 'react-apollo';

import { todoReducer, userReducer } from './reducers';

const client = new ApolloClient();

const store = createStore(
  combineReducers({
    todos: todoReducer,
    users: userReducer,
    apollo: client.reducer(),
  }),
  {}, // 初始状态
  compose(
      applyMiddleware(client.middleware()),
      // 如果您正在使用开发者工具扩展程序，您也可以在这里添加如下代码
      (typeof window.__REDUX_DEVTOOLS_EXTENSION__ !== 'undefined') ? window.__REDUX_DEVTOOLS_EXTENSION__() : f => f,
  )
);

ReactDOM.render(
  <ApolloProvider store={store} client={client}>
    <MyRootComponent />
  </ApolloProvider>,
  rootEl
)
```

如果要为客户端reducer使用不同的根密钥（而不是 `apollo`），请在创建客户端时使用 `reduxRootSelector: selector` 选项：

```js
const client = new ApolloClient({
  reduxRootSelector: state => state.differentKey,
});

const store = createStore(
  combineReducers({
    differentKey: client.reducer(),
  })
);
```

<h2 id="using-connect">使用连接</h2>

你可以继续使用`react-redux`的 `connect` 高阶组件将状态导入或导出组件。您可以连接GraphQL数据到您的组件之前或之后（或两者都）使用`graphql`：

```js
import React, { Component } from 'react';
import { graphql } from 'react-apollo';
import { connect } from 'react-redux';

import { CLONE_LIST } from './mutations';
import { viewList } from './actions';

const List = ({ listId, cloneList }) => (
  <div>List ID: {listId} <button onClick={cloneList}>Clone</button></div>
);

const withCloneList = graphql(CLONE_LIST, {
  props: ({ ownProps, mutate }) => ({
    cloneList() {
      return mutate()
        .then(result => {
          ownProps.onSelectList(result.id);
        });
    },
  }),
});
const ListWithData = withCloneList(List);

const ListWithDataAndState = connect(
  (state) => ({ listId: state.list.id }),
  (dispatch) => ({
    onSelectList(id) {
      dispatch(viewList(id));
    }
  }),
)(ListWithData);
```

这意味着您可以轻松地将变量传递到来自Redux状态的查询，或者调度依赖于服务器端数据的操作。
