---
date: "2024-06-13T20:58:47+08:00"
title: "Elasticsearch -- 基础操作"
tags: ["Database", "Elasticsearch"]
categories: "笔记"
description: ""
draft: false
searchHidden: false

showToc: true
TocOpen: true
hidemeta: false
comments: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: false
UseHugoToc: true
---

Elasticsearch 是一个高度可扩展的开源实时搜索和分析引擎，它允许用户在近实时的时间内执行全文搜索、结构化搜索、聚合、过滤等功能。<!--more-->Elasticsearch 基于 Lucene 构建，提供了强大的全文搜索功能，并且具有广泛的应用领域，包括日志和实时分析、社交媒体、电子商务等。

# 环境搭建

- 创建 compose.yaml 文件

```yaml
version: "3"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.9.1
    environment:
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - discovery.type=single-node
      - xpack.security.enabled=false
    networks:
      - elasticsearch
    ports:
      - 9200:9200
  kibana:
    image: docker.elastic.co/kibana/kibana:8.9.1
    networks:
      - elasticsearch
    ports:
      - 5601:5601
    environment:
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
    depends_on:
      - elasticsearch
networks:
  elasticsearch:
    driver: bridge
```

- 使用 docker-compose 运行

```sh
docker-compose up -d
```

- 访问 http://127.0.0.1:5601
  可能需要等待一会，Kibana 准备需要一些时间。进入页面后，从左侧菜单栏进入 Management-->DevTools 打开开发工具页面。后续我们就可以在页面左侧窗口中输入 curl 命令，点击 "▶" 符号发送请求后，在页面右侧窗口查看返回结果。

# API

## 查看健康状态

```
GET /_cat/health?v

epoch      timestamp cluster        status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1718103967 11:06:07  docker-cluster green           1         1      6   6    0    0        0             0                  -                100.0%

```

## 文档操作

### 创建文档

将 JSON 文档添加到指定的数据流或索引并使其可搜索。

- 如果索引不存在，则会创建默认配置的索引。索引相关内容将在后文详细介绍。
  我们已经通过索引一篇文档创建了一个新的索引。这个索引采用的是默认的配置，新的字段通过动态映射的方式被添加到类型映射。
- 如果目标是索引并且文档已经存在，则请求更新文档并递增其版本。

```
PUT /<target>/_doc/<_id>

POST /<target>/_doc/

PUT /<target>/_create/<_id>

POST /<target>/_create/<_id>
```

```
POST /movie_index/_create/1
{
    "id":1,
    "title":"a movie",
    "post_url":"post url",
    "tags":["action","sci_fic"],
    "desc":"this is a movie",
    "source_url":"source url"
}
```

### 判断文档是否存在

```
HEAD /movie_index/_doc/1
```

如果存在，Elasticsearch 返回 200 - OK 的响应状态码，如果不存在则返回 404 - Not Found。

### 获取文档

```
GET /movie_index/_doc/1
```

返回整个文档的内容，包括元数据。

### 获取数据

```
GET /movie_index/_source/1
```

### 获取指定字段

```
GET /movie_index/_source/1?_source=title,source_url
```

### 更新文档

```
POST /<index>/_update/<_id>

POST /movie_index/_update/1
{
  "doc": {
    "title": "good movie"
  }
}
```

### 批量获取

```
GET /_mget
GET /<index>/_mget
```

### 删除文档

```
DELETE /movie_index/_doc/1
```

### 批量操作

使用 `_bulk` API 进行批量操作，支持 `create`、`index`、`update`、`delete` 四种操作。

#### 语法格式

每个操作由两行组成：操作行 + 数据行（delete 除外），以换行符分隔。

```json
POST /_bulk
{ "index": { "_index": "movie_index", "_id": "1" } }
{ "title": "movie1", "tags": ["action"] }
{ "index": { "_index": "movie_index", "_id": "2" } }
{ "title": "movie2", "tags": ["comedy"] }
```

#### 批量创建

```json
POST /movie_index/_bulk
{ "create": { "_id": "10" } }
{ "title": "fastx", "tags": ["action"], "score": 9 }
{ "create": { "_id": "11" } }
{ "title": "interstellar", "tags": ["sci_fic"], "score": 9.5 }
{ "create": { "_id": "12" } }
{ "title": "toy story", "tags": ["comedy"], "score": 8.5 }
```

#### 批量更新

```json
POST /movie_index/_bulk
{ "update": { "_id": "10" } }
{ "doc": { "score": 9.2 } }
{ "update": { "_id": "11" } }
{ "doc": { "score": 9.3 } }
```

#### 批量删除

```json
POST /movie_index/_bulk
{ "delete": { "_id": "10" } }
{ "delete": { "_id": "11" } }
```

#### 注意事项

- 单次请求建议不超过 1000-5000 条数据，避免单条数据过大（单条不超过 100KB）
- `_bulk` 返回的 HTTP 状态码始终是 200，需要检查每条操作的 `status` 和 `error` 字段判断是否成功
- 同一操作行中的 `_index` 可以不同，支持跨索引批量操作

### 检索

```
GET /<target>/_search

GET /_search

POST /<target>/_search

POST /_search
```

- 查询 id=1 的文档。

```
GET /movie_index/_search
{
  "query": {
    "bool": {
      "filter":{
        "term":{"id": 1}
      }
    }
  }
}
```

- 查询 tags 包含 action 的文档。

```
GET /movie_index/_search
{
  "query": {
    "bool": {
      "filter":{
        "match_phrase":{
          "tags": "action"
        }
      }
    }
  }
}
```

- 查询 source 中包含 url 的文档。

```
GET /movie_index/_search
{
  "query": {
    "match_phrase": {
      "source_url": "url"
    }
  }
}
```

#### Bool 查询

Bool 查询是 ES 中最常用的组合查询，由四个子句组成：

| 子句       | 作用                   | 是否计算评分 |
| ---------- | ---------------------- | ------------ |
| `must`     | 必须匹配，类似 AND     | 是           |
| `should`   | 至少匹配一个，类似 OR  | 是           |
| `must_not` | 必须不匹配，类似 NOT   | 否           |
| `filter`   | 必须匹配，但不计算评分 | 否           |

##### must — 必须匹配

所有条件必须同时满足：

```json
GET /movie_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "movie" } },
        { "term": { "score": 9 } }
      ]
    }
  }
}
```

##### should — 至少满足一个

满足任一条件即可，匹配条件越多评分越高：

```json
GET /movie_index/_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "tags": "action" } },
        { "term": { "tags": "sci_fic" } }
      ]
    }
  }
}
```

##### must_not — 必须不匹配

排除满足条件的文档：

```json
GET /movie_index/_search
{
  "query": {
    "bool": {
      "must_not": [
        { "term": { "tags": "comedy" } }
      ]
    }
  }
}
```

##### filter — 过滤（不计算评分）

与 `must` 一样必须匹配，但跳过评分计算，性能更好。适合精确匹配和范围过滤：

```json
GET /movie_index/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "tags": "action" } },
        { "range": { "score": { "gte": 8 } } }
      ]
    }
  }
}
```

##### 组合使用

实际场景中经常组合使用多个子句：

```json
GET /movie_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "fast" } }
      ],
      "should": [
        { "term": { "tags": "action" } },
        { "term": { "tags": "sci_fic" } }
      ],
      "must_not": [
        { "term": { "tags": "comedy" } }
      ],
      "filter": [
        { "range": { "score": { "gte": 8 } } }
      ]
    }
  }
}
```

上例的含义：标题包含 "fast"，标签包含 action 或 sci_fic（匹配任一加分），排除 comedy 标签，且评分大于等于 8。

#### 常用查询类型

##### term — 精确匹配

用于 keyword、数值等结构化字段，不进行分词：

```json
{ "term": { "tags": "action" } }
```

##### match — 全文匹配

用于 text 字段，会进行分词后匹配：

```json
{ "match": { "title": "fast movie" } }
```

##### match_phrase — 短语匹配

要求分词后的词项按顺序相邻出现：

```json
{ "match_phrase": { "title": "fast movie" } }
```

##### range — 范围查询

支持数值和日期：

```json
{ "range": { "score": { "gte": 8, "lte": 10 } } }
```

##### wildcard — 通配符查询

支持 `*`（任意字符）和 `?`（单个字符）：

```json
{ "wildcard": { "title": "fast*" } }
```

##### exists — 字段存在性检查

```json
{ "exists": { "field": "score" } }
```

### 分页

#### from + size

基础分页方式，`from` 是起始位置（从 0 开始），`size` 是返回数量：

```json
GET /movie_index/_search
{
  "from": 0,
  "size": 10,
  "query": {
    "match": { "title": "movie" }
  }
}
```

查询第 2 页（每页 10 条）：

```json
{
  "from": 10,
  "size": 10,
  "query": { "match": { "title": "movie" } }
}
```

> 注意：`from + size` 不宜超过 10000（默认 `max_result_window`），超过后应使用 `search_after`。

#### search_after（深度分页推荐）

基于上一条结果的排序值继续查询，性能稳定，适合无限滚动场景：

```json
GET /movie_index/_search
{
  "size": 10,
  "sort": [
    { "score": "desc" },
    { "_id": "asc" }
  ],
  "query": {
    "match": { "title": "movie" }
  }
}
```

用上一页最后一条的 sort 值作为 `search_after` 参数查询下一页：

```json
{
  "size": 10,
  "search_after": [9.5, "11"],
  "sort": [{ "score": "desc" }, { "_id": "asc" }],
  "query": {
    "match": { "title": "movie" }
  }
}
```

#### scroll（大数据量导出）

适用于一次性导出大量数据，不支持实时翻页：

```json
// 第一次请求，创建 scroll 上下文
GET /movie_index/_search?scroll=1m
{
  "size": 100,
  "query": { "match_all": {} }
}

// 后续请求，使用返回的 _scroll_id
POST /_search/scroll
{
  "scroll": "1m",
  "scroll_id": "DXF1ZXJ5QW5kVGh1U3V..."
}
```

> 建议新代码优先使用 `search_after`，`scroll` 主要用于数据导出场景。

### 排序

使用 `sort` 参数指定排序字段和方向：

```json
GET /movie_index/_search
{
  "sort": [
    { "score": "desc" },
    { "created_at": "desc" }
  ],
  "query": {
    "match_all": {}
  }
}
```

- `asc`：升序
- `desc`：降序

对 keyword 字段排序：

```json
{
  "sort": [{ "title.keyword": "asc" }],
  "query": { "match_all": {} }
}
```

### 高亮

在搜索结果中标注匹配的关键词：

```json
GET /movie_index/_search
{
  "query": {
    "match": { "title": "fast movie" }
  },
  "highlight": {
    "fields": {
      "title": {},
      "desc": {}
    }
  }
}
```

返回结果中会包含 `highlight` 字段，用 `<em>` 标签包裹匹配文本：

```json
{
  "hits": {
    "hits": [
      {
        "_source": { "title": "fast and furious" },
        "highlight": {
          "title": ["<em>fast</em> and furious"]
        }
      }
    ]
  }
}
```

自定义高亮标签：

```json
{
  "highlight": {
    "pre_tags": ["<b>"],
    "post_tags": ["</b>"],
    "fields": {
      "title": {},
      "desc": {
        "fragment_size": 100,
        "number_of_fragments": 3
      }
    }
  }
}
```

- `fragment_size`：片段长度（仅对长文本生效）
- `number_of_fragments`：返回多少个高亮片段

### 获取数量

```
GET /<target>/_count

GET /movie_index/_count
{
  "query": {
    "match_phrase": {
      "title": "movie"
    }
  }
}
```

### 聚合

聚合分为两大类：

| 类型         | 说明                           | 常用聚合                            |
| ------------ | ------------------------------ | ----------------------------------- |
| **指标聚合** | 对数值字段计算统计值           | `avg`、`min`、`max`、`sum`、`count` |
| **桶聚合**   | 按条件分组，每个分组是一个"桶" | `terms`、`range`、`date_histogram`  |

#### 指标聚合

##### avg — 平均值

计算 score 字段的平均值：

```json
POST /movie_index/_search?size=0
{
  "aggs": {
    "avg_score": { "avg": { "field": "score" } }
  }
}
```

##### min / max — 最小值 / 最大值

```json
POST /movie_index/_search?size=0
{
  "aggs": {
    "min_score": { "min": { "field": "score" } },
    "max_score": { "max": { "field": "score" } }
  }
}
```

##### sum / count

```json
POST /movie_index/_search?size=0
{
  "aggs": {
    "total_score": { "sum": { "field": "score" } },
    "doc_count": { "value_count": { "field": "score" } }
  }
}
```

#### 桶聚合

##### terms — 按字段分组

统计每个标签有多少文档：

```json
POST /movie_index/_search?size=0
{
  "aggs": {
    "tags_agg": {
      "terms": { "field": "tags", "size": 10 }
    }
  }
}
```

返回结果：

```json
{
  "aggregations": {
    "tags_agg": {
      "buckets": [
        { "key": "action", "doc_count": 5 },
        { "key": "sci_fic", "doc_count": 3 },
        { "key": "comedy", "doc_count": 2 }
      ]
    }
  }
}
```

##### range — 范围分组

按分数区间分组：

```json
POST /movie_index/_search?size=0
{
  "aggs": {
    "score_ranges": {
      "range": {
        "field": "score",
        "ranges": [
          { "to": 6 },
          { "from": 6, "to": 8 },
          { "from": 8 }
        ]
      }
    }
  }
}
```

##### date_histogram — 按时间区间分组

按月统计文档数量：

```json
POST /movie_index/_search?size=0
{
  "aggs": {
    "by_month": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month"
      }
    }
  }
}
```

常用 `calendar_interval` 值：`minute`、`hour`、`day`、`week`、`month`、`quarter`、`year`

##### 嵌套聚合

桶聚合内嵌套指标聚合，如按标签分组后计算每组的平均分：

```json
POST /movie_index/_search?size=0
{
  "aggs": {
    "tags_agg": {
      "terms": { "field": "tags" },
      "aggs": {
        "avg_score": { "avg": { "field": "score" } }
      }
    }
  }
}
```

返回每个桶的平均分：

```json
{
  "buckets": [
    { "key": "action", "doc_count": 5, "avg_score": { "value": 8.8 } },
    { "key": "sci_fic", "doc_count": 3, "avg_score": { "value": 9.2 } }
  ]
}
```

## 索引操作

### 创建索引

现在我们需要对这个建立索引的过程做更多的控制：我们想要确保这个索引有数量适中的主分片，并且在我们索引任何数据之前，分析器和映射已经被建立好。
为了达到这个目的，我们需要手动创建索引，在请求体里面传入设置或类型映射，如下所示：

```
PUT /my_index
{
    "settings": { ... any settings ... },
    "mappings": {
        "type_one": { ... any mappings ... },
        "type_two": { ... any mappings ... },
        ...
    }
}
```

如果你想禁止自动创建索引，你可以通过在 config/elasticsearch.yml 的每个节点下添加下面的配置：

```yaml
action.auto_create_index: false
```

#### settings

配置项：

- number_of_shards: 每个索引的主分片数，默认值是 5。这个配置在索引创建后不能修改。
- number_of_replicas: 每个主分片的副本数，默认值是 1。对于活动的索引库，这个配置可以随时修改。
  ```
  PUT /my_temp_index/_settings
  {
    "number_of_replicas": 1
  }
  ```
- analysis: 来配置已存在的分析器或针对你的索引创建新的自定义分析器。中文一般使用 ik 分词器。
  ```
  PUT /spanish_docs
  {
      "settings": {
          "analysis": {
              "analyzer": {
                  "es_std": {
                      "type":      "standard",
                      "stopwords": "_spanish_"
                  }
              }
          }
      }
  }
  ```
  standard 分析器是用于全文字段的默认分析器，对于大部分西方语系来说是一个不错的选择。它包括了以下几点：
  - standard 分词器，通过单词边界分割输入的文本。
  - standard 语汇单元过滤器，目的是整理分词器触发的语汇单元（但是目前什么都没做）。
  - lowercase 语汇单元过滤器，转换所有的语汇单元为小写。
  - stop 语汇单元过滤器，删除停用词（对搜索相关性影响不大的常用词，如 a, the, and, is）

### 删除索引

```
DELETE /my_index

DELETE /index_one,index_two
DELETE /index_*
```

### 索引别名

别名为索引提供一个别名，可以在不修改代码的情况下切换索引。常用于零停机重建索引。

#### 创建别名

```json
POST /_aliases
{
  "actions": [
    { "add": { "index": "movie_v1", "alias": "movie" } }
  ]
}
```

#### 查看别名

```json
GET /movie/_aliases
GET /_aliases/movie
```

#### 切换别名（零停机重建索引）

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "movie_v1", "alias": "movie" } },
    { "add": { "index": "movie_v2", "alias": "movie" } }
  ]
}
```

#### 删除别名

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "movie_v1", "alias": "movie" } }
  ]
}
```

### Reindex

将数据从一个索引迁移到另一个索引，常用于修改 mapping 或分片数。

#### 基本用法

```json
POST /_reindex
{
  "source": { "index": "movie_v1" },
  "dest": { "index": "movie_v2" }
}
```

#### 带查询条件

只迁移符合条件的数据：

```json
POST /_reindex
{
  "source": {
    "index": "movie_v1",
    "query": {
      "term": { "tags": "action" }
    }
  },
  "dest": { "index": "movie_action" }
}
```

#### 远程集群迁移

```json
POST /_reindex
{
  "source": {
    "remote": {
      "host": "http://old-cluster:9200"
    },
    "index": "movie"
  },
  "dest": {
    "index": "movie"
  }
}
```

#### 控制写入速率

控制每批处理的文档数：

```json
POST /_reindex
{
  "source": { "index": "movie_v1" },
  "dest": { "index": "movie_v2" },
  "requests_per_second": 500
}
```

### \_cat API

用于查看集群状态的调试工具，添加 `?v` 显示表头。

#### 常用命令

```json
// 集群健康状态
GET /_cat/health?v

// 所有索引信息
GET /_cat/indices?v&s=index

// 节点信息
GET /_cat/nodes?v

// 分片分布
GET /_cat/shards?v

// 已分配的分片
GET /_cat/allocation?v
```

#### 指定格式和过滤

```json
// JSON 格式输出
GET /_cat/indices?format=json

// 按索引名过滤
GET /_cat/indices/movie*?v

// 按状态过滤
GET /_cat/indices?v&health=red
```

# Mapping

Mapping 类似于数据库中的表结构定义，用于定义索引中字段的名称、类型和行为。

## 查看 Mapping

```json
GET /movie_index/_mapping
```

返回当前索引的字段映射配置。

## 字段类型

### 核心类型

| 类型                                  | 说明                   | 示例                    |
| ------------------------------------- | ---------------------- | ----------------------- |
| `text`                                | 全文检索字段，会被分词 | `"title": "fast movie"` |
| `keyword`                             | 精确匹配字段，不分词   | `"tags": ["action"]`    |
| `long` / `integer` / `short` / `byte` | 整数类型               | `"id": 1`               |
| `double` / `float` / `half_float`     | 浮点数类型             | `"score": 9.5`          |
| `date`                                | 日期类型               | `"date": "2024-06-13"`  |
| `boolean`                             | 布尔类型               | `"active": true`        |

### text 和 keyword 的区别

- `text`：会被分词器处理，适合全文搜索（如搜索标题、描述）
- `keyword`：存储原始值，适合精确匹配、排序、聚合（如标签、状态）

一个字段可以同时设置两种类型：

```json
{
  "title": {
    "type": "text",
    "fields": {
      "keyword": {
        "type": "keyword"
      }
    }
  }
}
```

这样 `title` 字段支持全文搜索，`title.keyword` 支持精确匹配和聚合。

## 创建带 Mapping 的索引

```json
PUT /movie_index
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "id": { "type": "integer" },
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "tags": { "type": "keyword" },
      "score": { "type": "float" },
      "desc": {
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "source_url": { "type": "keyword" },
      "created_at": { "type": "date" },
      "active": { "type": "boolean" }
    }
  }
}
```

### 常用 mapping 参数

| 参数              | 说明                                        |
| ----------------- | ------------------------------------------- |
| `type`            | 字段类型                                    |
| `analyzer`        | 索引时使用的分词器                          |
| `search_analyzer` | 搜索时使用的分词器                          |
| `index`           | 是否对该字段建立索引，默认 true             |
| `doc_values`      | 是否启用列式存储，用于排序和聚合，默认 true |
| `null_value`      | 当字段值为 null 时的替代值                  |
| `fields`          | 多字段映射，一个字段多种索引方式            |

## 动态映射 vs 显式映射

- **动态映射**：索引文档时，ES 会自动推断字段类型并添加映射。方便但可能不精确（如数字被识别为 text）
- **显式映射**：创建索引时手动定义字段类型。推荐在生产环境中使用

## 更新 Mapping

已存在的字段类型不能修改（如 text 改为 keyword），需要重新建索引并 reindex 数据。

可以新增字段：

```json
PUT /movie_index/_mapping
{
  "properties": {
    "director": { "type": "keyword" }
  }
}
```

# Golang 客户端

使用 `go get` 命令下载客户端库文件。这是官方提供的库。

```sh
go get github.com/elastic/go-elasticsearch/v8@latest
```

## 连接

```go
// ES 配置
cfg := elasticsearch.Config{
	Addresses: []string{
		"http://localhost:9200",
	},
}

// 创建客户端连接
client, err := elasticsearch.NewTypedClient(cfg)
if err != nil {
	fmt.Printf("elasticsearch.NewTypedClient failed, err:%v\n", err)
	return
}
```

## 文档

### 批量操作

```go
// 批量创建
bulkReq := &types.BulkRequest{
	Operations: []types.BulkOperation{
		{
			Create: &types.BulkIndexOperation{
				Index: "movie",
				ID:    "1",
			},
		},
		{
			Document: Movie{
				Title: "fastx",
				Tags:  []string{"action"},
				Score: 9,
			},
		},
		{
			Create: &types.BulkIndexOperation{
				Index: "movie",
				ID:    "2",
			},
		},
		{
			Document: Movie{
				Title: "interstellar",
				Tags:  []string{"sci_fic"},
				Score: 9.5,
			},
		},
	},
}

resp, err := client.Bulk().Do(context.Background(), bulkReq)
if err != nil {
	return err
}
for _, item := range resp.Items {
	if item.Index != nil {
		fmt.Println(item.Index.Status)
	}
}
```

### 创建

```go
m := Movie{
		Title:  "title",
		Post:   "post",
		Tags:   []string{"tag1", "tag2"},
		Desc:   "desc",
		Source: "source",
	}
	resp, err := client.Index("movie").Document(m).Do(context.Background())
	if err != nil {
		return err
	}
	fmt.Println("result:", resp.Result)
	return nil
```

### 检索

```go
resp, err := client.Search().Index("movie").Query(&types.Query{
		Match: map[string]types.MatchQuery{"tags": {Query: "tag1"}},
	}).Do(context.Background())
	if err != nil {
		return err
	}
	for _, hit := range resp.Hits.Hits {
		fmt.Println(hit.Source_)
	}
	return nil
```

Bool 查询：

```go
resp, err := client.Search().Index("movie").Query(&types.Query{
	Bool: &types.BoolQuery{
		Filter: []types.Query{
			{Term: map[string]types.TermQuery{"tags": {Value: "action"}}},
			{Range: map[string]types.RangeQuery{
				"score": {Gte: &types.Number{Value: 8}},
			}},
		},
		Must: []types.Query{
			{Match: map[string]types.MatchQuery{"title": {Query: "fast"}}},
		},
		MustNot: []types.Query{
			{Term: map[string]types.TermQuery{"tags": {Value: "comedy"}}},
		},
	},
}).Do(context.Background())
if err != nil {
	return err
}
for _, hit := range resp.Hits.Hits {
	fmt.Println(hit.Source_)
}
```

带分页、排序和高亮：

```go
resp, err := client.Search().Index("movie").
	Query(&types.Query{
		Match: map[string]types.MatchQuery{"title": {Query: "fast movie"}},
	}).
	From(0).Size(10).
	Sort([]types.SortComputation{
		{Sort: &types.FieldSort{Field: "score", Order: types.SortOrderDesc}},
	}).
	Highlight(&types.Highlight{
		Fields: map[string]types.HighlightField{
			"title": {},
		},
	}).
	Do(context.Background())
if err != nil {
	return err
}
for _, hit := range resp.Hits.Hits {
	fmt.Println("source:", hit.Source_)
	fmt.Println("highlight:", hit.Highlight)
}
```

#### 更新

```go
resp, err := client.UpdateByQuery("movie").
		Query(&types.Query{MatchPhrase: map[string]types.MatchPhraseQuery{"source": {Query: "source"}}}).
		Do(context.Background())
	if err != nil {
		return err
	}
	fmt.Println(resp)
	return nil
```

### 查询数量

```go
resp, err := client.Count().Index("movie").
  Query(&types.Query{
    MatchPhrase: map[string]types.MatchPhraseQuery{
      "title": {Query: "fastx"},
    },
  }).Do(context.Background())
if err != nil {
  return err
}
fmt.Println(resp.Count)
return nil
```

### 删除

```go
resp, err := client.DeleteByQuery("movie").
		Query(&types.Query{Match: map[string]types.MatchQuery{"tags": {Query: "tag2"}}}).
		Do(context.Background())
	if err != nil {
		return err
	}
	fmt.Println(resp)
	return nil
```

## 索引

### 创建

```go
resp, err := client.Indices.
  Create("my-review-1").
  Do(context.Background())
if err != nil {
  fmt.Printf("create index failed, err:%v\n", err)
  return
}
fmt.Printf("index:%#v\n", resp.Index)
```

### 查找

```go
indices, err := client.Cat.Indices().Do(context.Background())
if err != nil {
  return err
}
for _, index := range indices {
  fmt.Println(*index.Index)
}
return nil
```

### 删除

```go
resp, err := client.Indices.Delete("movie").Do(context.Background())
if err != nil {
  return err
}
fmt.Println(resp.Acknowledged)
return nil
```
