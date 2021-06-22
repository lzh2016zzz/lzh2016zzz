
---
title: 关于 ElasticSearch的Update By Query
tags: 技术,elasticsearch
---
## 如何使用
Elasticsearch的`_update_by_query`操作允许elasticserch根据查询的结果批量获取文档,并通过脚本操作对这些文档进行更新.可以把它看成批量获取文档 + 批量更新的结合.

和`update`操作不同,`_update_by_query`不是原子操作,所有更新和查询失败都会导致`_update_by_query`操作中止.已经执行成功的更新操作仍然存在,不会被回滚.

下面是例子:

```json
POST member_info/_update_by_query
{
  "query": {
    "match": {
      "fid": "3388"
    }
  },
  "script": {
    "source": "ctx._source['dname'] = params['dname']",
    "params": {
      "dname": "有些问题本来就解决不了"
    }
  }
}
```

上面会先查出fid = 3388的所有文档,对这些文档分别执行脚本更新.

也可以稍微复杂点:

```json
POST member_info/_update_by_query
{
  "query": {
    "match": {
      "fid": "3388"
    }
  },
  "script": {
    //内嵌文档,存在即更新操作
    "source": """
    if(ctx._source.containsKey('member_relationships') && ctx._source['member_relationships'].size() > 0 )    {
  ctx._source.add(params['relationship']);
}else{
ctx._source['member_relationships'] = params['relationship'];
}
""",
"params": {
"relationship" : {"fid" : "3388","sid":"6625"}
}
}
}
```

## 关于坑
`_update_by_query`使用的是脚本更新,多个脚本同时更新一个文档时可能出现冲突.发生冲突时,只会保留其中一个的执行结果.其余请求则返回失败.

由于es是准实时的,每隔一段时间会把内存里的数据写入磁盘里,只有写入磁盘的文档才能被搜索到.

这意味着`_update_by_query`执行查询时得到的是索引的快照,如果在快照时间和处理索引请求时间之间发生更改,则会出现版本冲突.
默认刷盘间隔是1s,说白了,也就是1s内多次修改同一个文档就有可能出现冲突.

通过设置`maxRetries`可以在失败时自动重试:

```java
	UpdateByQueryRequest updateByQueryRequest = new UpdateByQueryRequest(index);
    //默认重试次数是11次,可见对elasticsearch来说有一定压力
	updateByQueryRequest.setMaxRetries(20);
```

也可以设置`refresh = true`,这个设置使得在调用`_update_by_query`后刷新索引,从而避免后续请求读到旧数据导致版本冲突:

```java
updateByQueryRequest.setRefresh(true);
```

重试或每次调用`_update_by_query`后刷新索引都要使用大量的资源,因此不建议将`_update_by_query`用于需要更新大量的文档场景.



## 性能优化

设置`slices`可以在多个分片上并行执行`_update_by_query`操作,提高执行效率:

```java
  updateByQueryRequest.setSlices(5);
```

脚本执行会占用大量的CPU资源,设置`requestsPerSecond`可以限制每个请求/秒执行的更新文档数:

```java
updateByQueryRequest.setRequestsPerSecond(100);
```
这个设置很重要,经过实测,仅仅每秒100+文档更新就会导致elasticsearch集群CPU占用率飙升(配置 : refresh = true,maxRetries = 20):

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/Unknown.png)


通过在程序上增加缓冲区,将在一段时间内对同一类文档的多个修改请求合并,可以减少请求次数,降低出现冲突的频率.




参考资料 :



[https://www.elastic.co/guide/en/elasticsearch/client/java-rest/master/java-rest-high-document-update-by-query.html](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/master/java-rest-high-document-update-by-query.html)



[https://www.elastic.co/guide/en/elasticsearch/reference/7.x/indices-refresh.html#refresh-api-desc](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/indices-refresh.html#refresh-api-desc)
