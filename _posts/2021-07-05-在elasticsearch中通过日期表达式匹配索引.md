---
tags: 
   - elasticsearch
article_header:
    type: overlay
    theme: dark
    background_color: '#203028'
    background_image:
      gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
      src: /assets/images/cover2.jpg
---

有时候我们需要按时间范围过滤文档.在elasticsearch里,支持通过日期表达式匹配索引名称.相比搜索所有索引/分片并过滤出结果,这样做的好处是可以减少扫描的文档数量并提高查询性能.绝大部分支持索引名称作为参数的api都支持日期表达式.

<!--more-->

日期表达式索引名称采用以下形式：

```bash
<static_name{date_math_expr{date_format|time_zone}}>
```

其中各个部分代表了:

- `static_name` : 索引的静态部分
- `date_math_expr` : 用于动态计算日期的数学表达式
- `date_format` : 日期的格式,这个是可选的,不传的话默认`YYYY.MM.dd`
- `time_zone` : 时区.不传默认utc时区,如果是中国时间要 + 8小时

需要注意日期格式中使用的小写和大写字母的意义.
例如：`mm`表示小时的分钟数，而`MM`表示一年中的月份。类似的情况还有`hh`表示1-12小时范围内的小时，与AM/PM组合使用，而`HH`表示0-23 24小时范围内的小时
{:.warning}


# 例子

下面的例子搜索了最近3天的日志:

```bash
# GET /<logstash-{now/d-2d}>,<logstash-{now/d-1d}>,<logstash-{now/d}>/_search
GET /%3Clogstash-%7Bnow%2Fd-2d%7D%3E%2C%3Clogstash-%7Bnow%2Fd-1d%7D%3E%2C%3Clogstash-%7Bnow%2Fd%7D%3E/_search
{
  "query" : {
    "match": {
      "test": "data"
    }
  }
}
```

比如现在的时间是2024年3月22日中午12点.utc

| 表达式                                   |      表示的值       |
| ---------------------------------------- | :-----------------: |
| `<logstash-{now/d}>`                     | logstash-2024.03.22 |
| `<logstash-{now/M}>`                     | logstash-2024.03.01 |
| `<logstash-{now/M{YYYY.MM}>`             |  logstash-2024.03   |
| `<logstash-{now/M-1M{YYYY.MM}}>`         |  logstash-2024.02   |
| `<logstash-{now/d{YYYY.MM.dd\|+12:00}}>` | logstash-2024.03.23 |

- `now`代表当前时间 , `/d`代表按天向下取整;所以`now/d` = 当前日期的12:00
- `/M`代表按月向下取整
- `{YYYY.MM}`是日期的格式.就是年.月
- 日期表达式还支持加减法. 所以`+12:00`就是3月23日
- url中的字符需要经过编码.比如`<`  = `%3C`



# 参考资料

[Date math support in index names  Elasticsearch Guide 7.3](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/date-math-index-names.html#date-math-index-names)
