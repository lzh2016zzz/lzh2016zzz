---
tags: 
   - elasticsearch
article_header:
    type: overlay
    theme: dark
    background_color: '#203028'
    background_image:
      gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
      src: /assets/images/cover1.jpg
---

elasticsearch提供`restful`风格的Api用于对索引进行增删改查操作.

<!--more-->

# API介绍

### 创建索引

```bash
POST /test_index
```

### 为索引添加新字段

```bash
PUT /test_index
{
  "properties": {
    "firstDtime" : {
      "type": "date"
    }
  }
}
```
不支持直接修改已经存在的属性的类型或者名称 ,需要新建一个字段,然后把旧字段的数据拷贝过去
{:.warning}

### 新增文档

```bash
POST /test_index/_doc
{
  "name": "测试一下~~~"
}
```

### 删除文档

```bash
DELETE /test_index/_doc/{索引id}
```

### 修改文档

实际上,elasticsearch的底层lucene索引不支持update.所以elasticsearch提供的`update api`其实是内部删除 + 更新的原子操作.好处是减少网络开销,并减少GET和index操作之间发生版本冲突的可能性.
{:.success}

```bash
PUT test_index/_doc/1
{
  "counter" : 1,
  "tags" : ["red"]
}
```

通过脚本对文档进行修改:

```bash
POST test_index/_update/1
{
  "script" : {
    "source": "ctx._source.counter += params.count",
    "lang": "painless",
    "params" : {
      "count" : 4
    }
  }
}
```

### 根据id批量查询

```bash
GET /test_index/_mget
{
  "ids" : ["1", "2"]
}
```

### 批量操作

在单个请求中执行多个索引或删除操作.可以降低网络开销并极大提高索引的速度.

```bash
POST /test_index/_bulk
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "create" : { "_index" : "test", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```
### 根据查询条件进行批量更新

只支持通过脚本更新,限制较多.建议不要将它用于大批量数据更新.

```bash
POST test_index/_update_by_query
{
  "script": {
    "source": "ctx._source.count++",
    "lang": "painless"
  },
  "query": {
    "term": {
      "user.id": "kimchy"
    }
  }
}
```

# 参考资料

[Document APIs](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html#docs)

