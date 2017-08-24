---
title: 查询分割
---

预取是使应用 UI 体验更快的简单方法。你可以根据鼠标事件预测可能需要的数据。
这个功能在浏览器上很强大，并且运行完好，但不能应用于移动设备。

改进 UI 体验的一个解决方案是使用片段来预先加载查询中的更多数据，但是加载大量（你可能永远不会向用户展示的）数据是非常昂贵的。

另一种方式是将巨大的查询分解成两个较小的查询：
- 第一个可以加载已经存在于 store 中的数据。这意味着它可以立即显示。
- 第二个查询可以加载尚未在 store 中的数据，并且必须立即从服务器中获取数据。

该方案的好处是可以让你不用获取太多数据，并且在服务器响应之前显示部分视图数据。

假设你有以下 schema：

```graphql
type Series {
  id: Int!
  title: String!
  description: String!
  episodes: [Episode]!
  cover: String!
}

type Episode {
  id: Int!
  title: String!
  cover: String!
}

type Query {
  series: [Series!]!
  oneSeries(id: Int): Series
}
```

并且你有两个视图：
1. 系列概述：所有系列的列表及其描述和封面
2. 系列详情：某个系列的详情视图及其描述，封面和剧集列表

系列概述的查询可能如下所示：

```graphql
query seriesOverviewData {
  series {
    id
    title
    description
    cover
  }
}
```

系列详情的查询可能如下所示：

```graphql
query seriesDetailData($seriesId: Int!) {
  oneSeries(id: $seriesId) {
    id
    title
    description
    cover
  }
}
```

```graphql
query seriesEpisodes($seriesId: Int!) {
  oneSeries(id: $seriesId) {
    id
    episodes {
      id
      title
      cover
    }
  }
}
```

通过为 `oneSeries` 字段添加一个[自定义解析器](cache-updates.html#cacheRedirect)（并且使用 dataIdFromObject 函数范式化缓存），可以从 store 立即解析数据，而无需服务器往返。

```javascript
import ApolloClient, { toIdValue } from 'apollo-client'

// ... 你的 NetworkInterface 声明
// 还有非常重要的：你的 dataIdFromObject 声明

const client = new ApolloClient({
  networkInterface,
  customResolvers: {
    Query: {
      oneSeries: (_, { id }) => toIdValue(dataIdFromObject({ __typename: 'Series', id })),
    },
  },
  dataIdFromObject,
})
```

实现两个查询的第二个视图的组件可能如下所示：

```jsx
import React, { PropTypes, } from 'react'
import { gql, graphql, compose, } from 'react-apollo'

const QUERY_SERIES_DETAIL_VIEW = gql`
  query seriesDetailData($seriesId: Int!) {
    oneSeries(id: $seriesId) {
      id
      title
      description
      cover
    }
  }
`

const QUERY_SERIES_EPISODES = gql`
  query seriesEpisodes($seriesId: Int!) {
    oneSeries(id: $seriesId) {
      id
      episodes {
        id
        title
        cover
      }
    }
  }
`
const options = ({ seriesId, }) => ({ variables: { seriesId, }, })

const withSeriesDetailData = graphql(QUERY_SERIES_DETAIL_VIEW, {
  name: `seriesDetailData`,
  options,
})

const withSeriesEpisodes = graphql(QUERY_SERIES_EPISODES, {
  name: `seriesWithEpisodesData`,
  options,
})

const withData = compose(
  withSeriesDetailData,
  withSeriesEpisodes
)

function SeriesDetailView({ 
  seriesDetailData: {
    loading: seriesLoading,
    oneSeries,
  },
  seriesWithEpisodesData: { 
    loading: episodesLoading,
    oneSeries: { episodes } = {},
  }
}) {
  return (
    <div>
      <h1>{seriesLoading ? `Loading...` : oneSeries.title}</h1>
      <img src={seriesLoading ? `/dummy.jpg` : oneSeries.cover} />
      <h2>Episodes</h2>
      <ul>
      {episodesLoading ? <li>Loading...</li> : episodes.map(episode => (
        <li key={episode.id}>
          <img src={episode.cover} />
          <a href={`/episode/${episode.id}`}>{episode.title}</a>
        </li>
      ))}
      </ul>
    </div>
  )
}

const SeriesDetailViewWithData = withData(SeriesDetailView)

SeriesDetailViewWithData.propTypes = {
  seriesId: PropTypes.number.isRequired,
}

export default SeriesDetailView

```

不巧的是，如果用户现在先访问第二个视图，而没有访问过第一个视图，这将导致两个网络请求（因为第一个查询的数据不在 store 中）。通过使用 [`BatchedNetworkInterface`](http://dev.apollodata.com/core/apollo-client-api.html#BatchedNetworkInterface)，这两个查询可以合并到同一个网络请求中，然后才发送给服务器。
