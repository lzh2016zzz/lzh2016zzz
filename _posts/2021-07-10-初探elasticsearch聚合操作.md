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

根据官方定义,elasticsearch的聚合分为三类: `Metric`,`Bucket`,`Pipeline` 

原谅我英语不好,实在是找不到合适的翻译.用通俗的话来描述:

1. [Metric](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics.html) 根据字段的值做计算.类似`sum`,`avg`,`min`,`max`这样的
2. [Bucket](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket.html) 根据字段的值或范围进行分组处理.每种类型都会被放在一个叫做存储桶(`bucket`)的数据结构里.类似MySQL的`group by`
3. [Pipeline](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-pipeline.html) 这个类型比较特殊.不是根据文档做聚合,而是通过其他聚合操作的结果进行二次聚合

下面简单介绍一下这几种聚合API的用法.


# 分组

## 根据字段的值进行分组

通过`terms`进行聚合查询,返回的结果类似MySQL的`group by`.DSL大概是这样: 

```bash
 ...
 "aggregations": {
        "template_item_code": {
            "terms": {
                "field": "template_item_code"
            }
        }
 }
```

<!--more-->

`terms`可以根据字段的值对扫描到的文档进行分类.比如我们现在有三种类型的流水数据: `1001`,`1002`,`1003`.
elasticsearch就会创建三个bucket,分别存放这三种类型的数据的信息.

`key`是流水类型,默认情况下会返回这三种流水类型的文档数量,也就是`doc_count`:

```json
...
"aggregations" : {
    "template_item_code" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "1001",
          "doc_count" : 1325
        },
        {
          "key" : "1002",
          "doc_count" : 675
        },
        {
          "key" : "1003",
          "doc_count" : 225
        }
      ]
    }
  }
```

默认下,elasticsearch的`terms`只会返回10条数据.通过`size`可以设置返回的条目数量
{:.warning}

## 根据分组二次聚合

有时候我们想要对`terms`分组的数据进行二次聚合.

比如上面的例子,还需要分别对每种账本类型的交易金额进行`求和`操作,可以这样写:

```json
   "aggregations": {
        "template_item_code": {
            "terms": {
                "field": "template_item_code"
            },
            "aggregations": {
                "money_sum": {
                    "sum": {
                        "field": "trade_money"
                    }
                }
            }
        }
}
```

返回的数据里包含了`money_sum`,如果数值太大,`value`就会自动变成科学计数法:

```json
...
"aggregations" : {
    "template_item_code" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 1001,
          "doc_count" : 1325,
          "money_sum" : {
            "value" : 209999912
          }
        },
        {
          "key" : 1002,
          "doc_count" : 675,
          "money_sum" : {
            "value" : 2.01852418E10
          }
        }]
}
```

## 普通范围聚合

有时候我们需要根据时间范围对文档进行聚合操作.比如统计最近三个月内每个月的订单数量.这时候就可以用上范围聚合了:


```bash
GET /completeorder/_search
{
    "aggregations": {
    "time_ranges": {
      "range": {
        "field": "rectime",
        "ranges": [
          { "from": "2021-03-01 00:00:00","to": "2021-03-31 23:59:59"},
          { "from": "2021-04-01 00:00:00","to": "2021-04-30 23:59:59"},
          { "from": "2021-05-01 00:00:00", "to": "2021-05-31 23:59:59"}
        ]
      }
    }
  }
}
```

不带时区时,默认是UTC时区.中国时间要快8小时
{:.warning}



每个月的订单数量都会被作为一个存储桶返回:

```json
...
  "aggregations" : {
    "time_ranges" : {
      "buckets" : [
        {
          "key" : "2021-03-01T00:00:00+0000-2021-03-31T23:59:59+0000",
          "from" : 1.6145568E12,
          "from_as_string" : "2021-03-01T00:00:00+0000",
          "to" : 1.617235199E12,
          "to_as_string" : "2021-03-31T23:59:59+0000",
          "doc_count" : 1186
        },
        {
          "key" : "2021-04-01T00:00:00+0000-2021-04-30T23:59:59+0000",
          "from" : 1.6172352E12,
          "from_as_string" : "2021-04-01T00:00:00+0000",
          "to" : 1.619827199E12,
          "to_as_string" : "2021-04-30T23:59:59+0000",
          "doc_count" : 524
        },
        {
          "key" : "2021-05-01T00:00:00+0000-2021-05-31T23:59:59+0000",
          "from" : 1.6198272E12,
          "from_as_string" : "2021-05-01T00:00:00+0000",
          "to" : 1.622505599E12,
          "to_as_string" : "2021-05-31T23:59:59+0000",
          "doc_count" : 700
        }
      ]
    }
  }
```

## IP范围聚合

elasticsearch对IP范围聚合做了优化,在对用户请求来源做统计时非常有用:
```bash
GET /ip_addresses/_search
{
  "size": 10,
  "aggs": {
    "ip_ranges": {
      "ip_range": {
        "field": "ip",
        "ranges": [
          { "to": "10.0.0.5" },
          { "from": "10.0.0.5" }
        ]
      }
    }
  }
}
```



# 计算

## 简单的求和操作

dsl可以这样写:

```bash
GET /pft_trade_journal/_search
{
  "query": {
    "match_all": {}
  },
  "aggregations":{
  	"total_dmoney" : {
  		"sum" : {
  			"field" : "dmoney"
  		}
  	}
  }
}
```

这个聚合查询返回的是`dmoney`字段的和,格式是这样的 :

```bash
 ...
"aggregations": {
  "total_dmoney": {
    "value": 11992318812928
}
```

`total_money`是聚合的名称,`value`是聚合的结果.


## 求最大值/最小值/平均值

```json
...
"aggregations":{
  	"max_pay_money" : {
  		"max" : {
  		//"min" : {
  		//"avg" : {
  			"field" : "pay_money"
  		}
  	}
}
```

## 求加权平均值

`weight`是权重字段

```bash
...
"aggregations": {
    "weighted_grade": {
      "weighted_avg": {
        "value": {
          "field": "grade"
        },
        "weight": {
          "field": "weight"
        }
      }
    }
  }
```
## 计算百分位

```bash
...
"aggs": {
    "load_time_outlier": {
      "percentiles": {
        "field": "load_time" 
      }
    }
  }
}
```

默认情况下，百分位数度量将生成一系列百分位数：`[1%、55、25%、50%、75%、95%、99%]`,也可以自己指定百分位.
```bash
{
  ...
 "aggregations": {
    "load_time_outlier": {
      "values": {
        "1.0": 5.0,
        "5.0": 25.0,
        "25.0": 165.0,
        "50.0": 445.0,
        "75.0": 725.0,
        "95.0": 945.0,
        "99.0": 985.0
      }
    }
  }
}
```

其他(Metric)类型的聚合API使用方式大同小异,就不再赘述了.对了,Metric还支持脚本操作.可以用于复杂的数据计算.
{:.success}

## 脚本聚合运算 (7.18补充)

使用`scripted_metric`可以通过编写脚本对文档进行复杂的聚合运算,语法和`groovy`类似,兼容`java`语法,下面简单介绍一下它的用法.

下面的聚合从文档中取出内嵌数组字段`aids_splits.sale_money`并进行求和操作,返回值存储在`state.moneys`里:

```json
...
"aggs": {
    "totalMoney": {
      "scripted_metric": {
        "init_script": "state.moneys = 0",
        "map_script": """
        	Map aids_splits = params.aids_split;
        	state.moneys += aids_split.sale_money
        """,
        "combine_script": "state",
        "reduce_script": """
          def result = [:];
          result.moneys = 0;
          for(state in states){
            result.moneys +=state.moneys
          }
          result
        """
      }
    }
  }
```

返回值是这样的,因为数字太大所以变成了科学计数法:

```json
...
"aggregations" : {
    "totalMoney" : {
      "value" : {
        "moneys" : 9.0969404096E10
      }
    }
  }
```

脚本分为四个阶段: 

- **init_script** : 初始化脚本,每个文档执行一次,用来做一些初始化的工作.在上面的例子里,将`state.moneys`的默认值设置为0
- **map_script**: 映射脚本,每个文档执行一次.用来从文档中提取聚合需要的数据,在上面的例子里,从文档中提取`aids_splits`字段,并将`sale_money`存储在state中.
- **combine_script** : 合并脚本,每个分片会通过执行这个脚本分别将收集到的数据做聚合处理.上面的例子里啥也没干.直接返回了state
- **reduce_script** : 归约脚本.用于将每个分片收集到的结果做合并为一个结果.上面的例子里将每个分片收集到的结果进行了求和操作.

使用脚本聚合会导致搜索速度变慢,集群负载飙升,应该尽量避免作为常规查询手段使用.尽量通过合理的索引结构设计完成需求.
{:.error}


# 数据误差问题

使用elasticsearch做聚合操作时,结果可能带有误差. 举个例子,比如我们要获取每个分组里`money`最多的5个用户.

这时,客户端向服务端发起请求,假设索引有5个分片,那么服务端会向这5个分片分别发起请求获取`money`前5的用户. 再把这5个分片的数据组装起来,选出`money`最多的5个用户返回给客户端.

这样是有问题的,统计结果可能出现误差.原因是数据分布可能不均匀,比如某个分片里存在`money`前7名的用户,多出来的2个用户没排到前5,所以没在最后的结果里出现.这就导致最后结果少计算了2个用户.

## 解决方案

通过设置`size`和`shard_size`可以解决这个问题.

- size : 汇总以后返回的文档数量
- shard_size : 每个分片返回的文档数量

如果我们的分片数量是5,想取money数量前5名的用户,那么`shard_size`至少应该设置为5 * 5 = 25,才能保证不会出现误差的情况.
当 `shard_size` 设置为小于5 ~ 25时,可以减少误差.此时需要在精度和性能之间做取舍.






# 参考资料

 [Elasticsearch Guide 7.13 Aggregations](https://github.com/elastic/elasticsearch/edit/7.13/docs/reference/aggregations.asciidoc)

