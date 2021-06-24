---
tags: 
  elasticsearch
---

本篇翻译的是elasticsearch7.0版本官方文档的通用配置.以下配置可以用于优化对elasticsearch的请求,且适用于所有的Rest API,这对使用elasticsearch是很有帮助的

原文 : [https://www.elastic.co/guide/en/elasticsearch/reference/7.0/common-options.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.0/common-options.html)

<!--more-->

# 通用配置

以下配置适用于所有的Rest API

## 返回值格式化

在任何请求后附加query参数`pretty=true`时，返回的JSON将被格式化,具有更好的可读性.另外还可以设置`format=yaml`,这样结果将以yml格式返回.

## 使返回值具有更好的可读性

在query参数中附加`human=true`可以将返回值中的时间/字节数等单位转换为可读性更好的值.比如 : 

- EXISTS_TIME_IN_MILIS : 3600000 => EXISTS_TIME : "1h" 
- SIZE_IN_BYTES : 1024 => SIZE : "1KB"

当返回的结果是供人直接阅读,而不是由机器进行处理时,这个配置是相当有用的.

## 日期数学表达式

在elasticsearch中,日期表达式是很常见的.大部分接收日期作为参数的对象都支持传入日期数学表达式,如`gt/lt` 或`range`查询.

常见的值:

- +1h ，表示加上一个一个小时

- -1d，表示减去一天

- /d，表示向一天取整

支持的单位也很多:

| 单位 | 时间 |
| ---- | ------- |
| `y`  | 年   |
| `M`  | 月   |
| `w`  | 周   |
| `d`  | 天   |
| `h`  | 小时 |
| `H`  | 小时 |
| `m`  | 分   |
| `s`  | 秒 |

假设现在是2001-01-01 12:00:00，那么：

| 表达式                | 意义                                                        |
| --------------------- | ----------------------------------------------------------- |
| `now+1h`              | 当前时间 + 1小时 :  `2001-01-01 13:00:00`                   |
| `now-1h`              | 当前时间 - 1小时 :  `2001-01-01 11:00:00`                   |
| `now-1h/d`            | 减去一小时,然后向下舍入到当天0时0点0分: `2001-01-01 00:00:00` |
| `2001.02.01\|\|+1M/d` | 向下入到当天0时0点0分,然后再+一个月: `2001-03-01 00:00:00`  |

## 定制需要的返回结果

有时候我们不需要返回所有的项目,因为这样会浪费一定的服务器资源和带宽.所有的api都支持通过`filter_path`参数用来定制返回结果.需要定制多个参数时用`,`隔开

举个例子,如果我们只希望返回`took`,`hits.hits`,`hits._id`,可以像下面这样:

```console
GET /_search?q=elasticsearch&filter_path=took,hits.hits._id
```
返回值:
```js
{
  "took" : 3,
  "hits" : {
    "hits" : [
      {
        "_id" : "0"
      }
    ]
  }
}
```

它还支持`*`通配符来匹配任何字段或字段名称的一部分:

```console
GET /_cluster/state?filter_path=metadata.indices.*.stat*
```

或者使用`**`来匹配中间的多级路径:

```console
GET /_cluster/state?filter_path=routing_table.indices.**.state
```
逻辑和`spring mvc`的`**`基本一致

## 扁平化

设置`flat_settings=true`时,会将结果集中的内嵌对象转换为平铺的结构:

```json
GET twitter/_settings?flat_settings=true
# 扁平化的后的返回值,可以看到所有的返回值都在同一级内
{
  "twitter" : {
    "settings": {
      "index.number_of_replicas": "1",
      "index.number_of_shards": "1",
      "index.creation_date": "1474389951325",	
      "index.uuid": "n6gzFZTgS664GUfx0Xrpjw",
      "index.version.created": ...,
      "index.provided_name" : "twitter"
    }
  }
}
#正常的返回值,嵌套结构
{
  "twitter" : {
    "settings" : {
      "index" : {
        "number_of_replicas": "1",
        "number_of_shards": "1",
        "creation_date": "1474389951325",
        "uuid": "n6gzFZTgS664GUfx0Xrpjw",
        "version": {
          "created": ...
        },
        "provided_name" : "twitter"
      }
    }
  }
}
```

默认的`flat_settings`值是false

## 参数

所有http url参数都遵循使用下划线代替驼峰的规范.

## 布尔值类型

所有参数（包括query参数json体）都支持传入字符串"false"作为值false，"true"作为值true,传入其他任意值都将得到一个异常.

## 数字类型

支持传入字符串

## 时间单位

每当需要指定时间范围时,如`timeout`,都必须指定单位:

| 单位     | 时间范围 |
| -------- | -------- |
| `d`      | 天       |
| `h`      | 小时     |
| `m`      | 分       |
| `s`      | 秒       |
| `ms`     | 毫秒     |
| `micros` | 微秒     |
| `nanos`  | 纳秒     |

## 字节大小单位

每当需要指定数据的字节大小时，例如在设置缓冲区大小参数时，该值必须指定单位，例如:`10kb` ,`1mb`

注意:这些单元使用1024的幂，因此1kb表示1024字节.支持的单位有(感觉不用翻了吧)：

| 单位 | 大小      |
| ---- | --------- |
| `b`  | Bytes     |
| `kb` | Kilobytes |
| `mb` | Megabytes |
| `gb` | Gigabytes |
| `tb` | Terabytes |
| `pb` | Petabytes |

## 距离单位

如果需要指定距离，例如[Geo-distance](https://www.elastic.co/guide/en/elasticsearch/reference/7.0/query-dsl-geo-distance-query.html) 中的`distance`参数，默认单位为米,也可以指定为其他单位

## 模糊匹配

有的时候一些需要允许一定的模糊度，比如检索hallo可以查询到hello，这就要支持模糊查询。模糊查询可以使用fuzziness参数，它有点像range：

```
-fuzziness <= fieldValue <= +fuzziness
```

并且可以设置一定的模糊度，比如：

- 0,1,2 设置它的编辑距离（[levenshtein distance，wiki](https://en.wikipedia.org/wiki/Levenshtein_distance)）
- AUTO，如果设置Auto，那么会根据字符串的长度而改变

比如，长度为:

- `0..2`，必须完全匹配
- `3..5`，可以有一个编辑距离的模糊度
- `>5`，可以有两个编辑距离的模糊度

## 异常时返回stacktrace

默认情况下，当请求返回错误时，Elasticsearch不会返回错误的堆栈跟踪.通过设置`error_trace=true`可以可以开启这个特性:

```bash
POST /twitter/_search?size=surprise_me&error_trace=true
```

## 查询的请求体
除了POST请求外，其他的请求不支持请求体,这个时候如果要执行查询，可以把参数放在Url后面.

