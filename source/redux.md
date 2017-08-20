---
title: 与 Redux 集成
---

默认情况下，Apollo 客户端会创建自己的内部 Redux store 来管理查询及其结果。如果你已经在应用的其余部分里使用了 Redux，则可以将客户端与现有 store 集成。

> 注意：虽然这将使 Apollo 客户端将其数据保存在同一个 store 中，但你仍然应该使用 [graphql 容器](api-graphql.html) 将数据附加到 UI 中。如果要在同一个组件中使用 Redux 和 Apollo 的状态，则需要_同时_使用 react-apollo 的 `graphql` 和 react-redux 的 `connect`。

这将让你更好地跟踪应用中发生的不同事件，以及客户端和服务器端数据如何交错更改。在使用像 [Redux 开发者工具](https://github.com/zalmoxisus/redux-devtools-extension)这样的工具时将会更加自然。

<h2 id="creating-a-store">创建一个 store</h2>

如果你想使用自己的 store，你需要从 Apollo 客户端实例中为其传递 reducer 和中间件；你可以直接将 store 传递给你的 `ApolloProvider` 中：

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
      // 如果你正在使用开发者工具扩展程序，你可以在这里添加如下代码
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

如果要为客户端 reducer 使用不同的根键（而不是 `apollo`），请在创建客户端实例时使用 `reduxRootSelector: selector` 选项：

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

<h2 id="using-connect">使用 connect</h2>

你可以继续使用 `react-redux` 的 `connect` 高阶组件将状态导入或导出组件。你可以连接 GraphQL 数据到你的组件之前或之后（或两者都）使用 `graphql`：

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

这意味着你可以轻易地将来自 Redux 状态的变量传递给你的查询，或者 dispatch 依赖于服务端数据的 action。
