---
tags: 
  elasticsearch
---

es支持两种查询方式,`请求体查询` 和 `query参数查询`.同时指定两种查询方式时,只有query参数查询生效.

<!--more-->

## query参数查询

```bash
GET /my_index/_search?q=*&from=40&size=20
```
其中`my_index`是查询索引的名称,`q=*`是搜索条件,表示查询所有内容

## 请求体查询(推荐)

```bash
GET /my_index/_search
{
  "query": {
    "match_all": {}
  }
  ,"from": 40,
  "size": 20
}

```

`query`定义了一个查询,`match_all`定义了搜索条件,表示查询所有内容,`from`和`size`等价SQL中的`limit`和`offset`,上面的命令请求了索引里的第40~60条的文档.

这种方式会把查询条件放入request-body中,特点是易于理解,支持更加复杂的查询逻辑.实际业务中,大部分情况下会使用这种方式.

## 返回的内容
```json
{
  "took" : 11,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 60,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "my_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
           "id" : "111",
           "name" : "bob"
        }
      },

      {
        "_index" : "my_index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
           "id" : "222",
           "name" : "lucas"
        }
      }
      //...略过其他数据
    ]
  }
}
```

返回的内容大概是这样的:

- took : 执行查询花费的时间,单位毫秒

- timed_out: 是否超时

- _shards: 分片信息,包括总分片数,成功分片数,失败分片数等

- hits: 搜索的结果,total是满足查询条件的总记录数,hits -> hits里是实际返回的记录数量

- _score: 文档评分信息,跟文档的排名有关.参考搜索引擎的排名就很容易理解


默认情况下es一次最多返回10条数据,通过`size`参数可以指定返回的最大记录数量

## Query DSL
Elasticsearch提供了基于JSON的DSL（域特定语言）来定义查询.

分页查询第20-40条数据:
```json
GET /my_index/_search
{
  "query": {
    "match_all": {}
  }
  ,"from": 20,
  "size": 20
}

```

指定排序方式为`fid desc`:

```json
GET member_info/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "fid": {
        "order": "desc"
      }
    }
  ]
}

```

通过match过滤`fid = 3385`的文档:

```json
GET member_info/_search
{
  "query": {
    "match": {
      "fid": "3385"
    }
  }
}
```

返回所有的字段会造成网络开销和服务器资源的浪费,通过`source`可以指定返回特定字段:

```json
GET member_info/_search
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}
```

ES提供了bool查询，可以把很多小的查询组成一个更为复杂的查询，比如查询同时包含mill和lane的文档:
```json
GET member_info/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

修改bool参数为should,可以改为查询包含mill或者lane的文档：

```json
GET member_info/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

有时候我们只需要获取满足条件的文档数量,可以使用count:

```json
GET member_info/_count
{
  "query": {
    "match": {
      "source": "24"
    }
  }
}
```

