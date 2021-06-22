
---
title: Elasticsearch深度分页处理
tags: 技术,elasticsearch
---
从es拉取数据时,如果我们想要查询前10条数据.正常流程是:

1. 客户端发送请求给某个节点

2. 节点将请求转发给分片,查询每个分片上的前10条数据

3. 分片将数据返回给节点,整合数据,提取前10条

4. 返回请求给客户端

如果要查询10-20条数据,就需要用到分页查询.

## 通过from + size进行浅分页

浅分页的概念是我自己定义的.原理比较简单,就是先查出前20条数据,然后丢弃前10条.这样其实白白浪费了前10条数据.


这样使用:

```
GET completeorder/_search
{
  "query": {
    "match_all": {}
  }
  ,
  "from": 10,//偏移
  "size": 10//返回的条数
}
```

之前做过测试,**偏移越大,分页响应时间越长,占用的资源也越多**.

默认情况下,es限制from + size不能 > 10000是一种保护措施.可以通过修改`max_result_window`提高这个阈值.

## scroll分页

相对于from + size 分页来说,使用scroll分页可以模拟一个游标.记录当前读取的文档位置.

这个分页用法会在 es服务端维护一个当前索引的快照信息,在此快照创建以后任何新增的数据,都无法在这个快照中查询到,所以这种分页用法不能用于实时查询数据,而是用于一次查询大量的数据.

它相对于from/size来说,并不是查出所有数据然后剔除不要的部分,而是记录一个读取的位置,保证下一次能快速读取.

这样使用:

```
GET completeorder/_search?scroll=1m
{
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    }
}
```

这个查询会返回一个`_scroll_id`,通过这个id可以继续查询

```
GET completeorder/_search?scroll=1m&scroll_id=nRYS0o4bG5RSW1kaXdjRHRQVC1rQTsyMjg5OnZ0WEtKOGxuUUltZGl3Y0R0UFQta0E7MjI4NDp2dFhLSjhsblFJbWRpd2NEdFBULWtBOzIyODU6dnRYS0o4bG5RS
```

游标会把当前索引的快照信息保存在内存里.启用游标查询可以通过在查询的时候设置参数 `scroll` 的值为我们期望的游标查询的过期时间。这个过期时间的参数很重要，因为保持这个游标查询窗口需要消耗资源，所以我们期望如果不再需要维护这种资源就该早点儿释放掉。 设置这个超时能够让 Elasticsearch 在稍后空闲的时候自动释放这部分资源.

## search_after分页

`Scroll Api`效率高,但维护索引快照上下文成本很高,建议不要将其用于实时用户请求.

相对于Scroll Api而言,`search_after`通过提供一个活动游标来避免这个问题,它的原理是使用上一页的结果来帮助快速检索下一页的数据.

`seach_after`和`scroll api` 很相似,但和它不同的是,`search_after`是无状态的,它总是会根据最新版本的索引内容进行解析.因此,排序顺序可能会根据索引的内容变化发生变化.


这样使用:

```
GET completeorder/_search
{
  "size": 10,
  "query": {
    "match_all": {}
  }
  ,"sort": [
    {
      "ordertime": {
        "order": "desc"
      }
    }
  ]
}
```

上面的查询返回的每个文档都会带上一个`sort`数组,这些`sort`数组可以用于传递给`search_after`参数以便抓取下一页的数据:

```
{
  "took" : 255,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
      ...略过9条数据
      {
        "_index" : "completeorder",
        "_type" : "_doc",
        "_id" : "62273485216668",
        "_score" : null,
        "_source" : {
              ...略
        },
        "sort" : [
          1622734852000
        ]
      }
    ]
  }
}

```

比如，我们可以使用最后的一个文档的sort排序值，将它传递给 search_after 参数,用来查出下一页的数据:

```
GET completeorder/_search
{
  "size": 10,
  "query": {
    "match_all": {}
  },
  "search_after": [1622734852000],
  ,"sort": [
    {
      "ordertime": {
        "order": "desc"
      }
    }
  ]
}
```


参考资料: 

[https://www.elastic.co/guide/en/elasticsearch/reference/6.8/search-request-search-after.html](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/search-request-search-after.html)

[https://www.elastic.co/guide/en/elasticsearch//reference/current/scroll-api.html](https://www.elastic.co/guide/en/elasticsearch//reference/current/scroll-api.html)


