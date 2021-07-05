---
tags: 
   - elasticsearch
---

在elasticsearch中搜索中文数据时.如果使用默认的分词器,在搜索时会出现输入一个词或一段话却搜索不到结果,或者只能通过一个字去匹配的情况.这是因为默认分词器只支持将中文单词分割成一个一个字,通过引入中文分词器可以解决这个问题.

# 分词器和倒排索引

分词器顾名思义,就是将一段话分成一个个的词,这个概念在搜索引擎中很重要.中文分词则更加复杂,比如`中华人民共和国人民`,不能简单地当成一个词,也不能粗暴的分成`中华人民共和国`和`人民`,`共和国`和`中华`这些也都算一个词!

一般做法是做一个标准的词典,关键词都在这个词典里面,然后按照几种规则去查找有没有关键词,比如:

- 正向最大匹配(从左到右)

- 逆向最大匹配(从右到左)

- 最少切分

- 双向匹配（从左扫描一次，从右扫描一次）

<!--more-->

分词器切分出来的单词会被用于倒排索引(Inverted index).倒排索引实现的是`单个词` 到`一个或者一组文档`的映射.通过倒排索引可以通过单词快速地检索到包含某个词的文档.

倒排索引的结构类似这样 : 


| 关键词 | 文档                              |
| ------ | --------------------------------- |
| 中华   | 中华人民共和国,中华人民共和国国民 |
| 国民   | 中华人民共和国国民                |
| 共和国 | 中华人民共和国,中华人民共和国国民 |

通过简单的映射就可以检索到需要的文档.和后缀树这样的数据结构相比,相当于用存储空间换取了更快的查询速度.

## text类型

在elasticsearch中, `text`类型是最常见的类型.开启动态索引后,如果不是数字或者标准的日期格式,都会以`text`类型进行存储.

默认情况下,elasticsearch会使用默认分词器对`text`类型进行解析.如果不希望elasticsearch对字符类型进行解析,而是希望把它作为一个整体,应该使用`keyword`类型.

# 中文分词插件

通常国内云服务商提供的elasticsearch默认会安装 IK中文分词插件（英文名为analysis-ik）.这也是目前使用最广泛的中文分词插件.如果是本地部署elasticsearch的话就需要自己安装了.

## 安装插件

首先确保docker已经启动
{:.success}

进入容器:
```bash
docker exec -it elasticsearch /bin/bash
```
安装ik分词器  [GitHub](https://github.com/medcl/elasticsearch-analysis-ik){:.button.button--success.button--pill}
```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.13.2/elasticsearch-analysis-ik-7.13.2.zip
```
版本号不是越新越好.每个elasticsearch版本对应的ik分词器版本都有不同.在Github上作者罗列出了elasticsearch版本对应的ik分词器版本,下载对应的版本即可.
{:.warning}


安装完以后会显示这个,大意是安装ik分词器成功了,需要重启elasticsearch生效.

```bash
-> Installed analysis-ik
-> Please restart Elasticsearch to activate any plugins installed
```



## 模拟测试

### 测试ik分词器


analysis-ik主要提供两种粒度的Analyzer:

- ik_max_word：会将文本做最细粒度的拆分；尽可能多的拆分出词语
- ik_smart：会做最粗粒度的拆分；已被分出的词语将不会再次被其它词语占有

新增一个文档:

```bash
POST /testdatas/_doc
{
  "text": "中国人民的爱国主义和自力更生精神，是社会主义现代化建设的强大力量"
}
```

通过`_analyze`执行查询分析,看看分词器是否生效了:

```bash
GET /_analyze
{
 //"analyzer" : "ik_max_word",
  "analyzer" : "ik_smart",
    "text" : "中国人民共和国"
}
```

`ik_max_word`的分词效果如下,对文本做了最细粒度的拆分:

```json
{
  "tokens": [
    {
      "token": "中国人民",
      "start_offset": 0,
      "end_offset": 4,
      "type": "CN_WORD",
      "position": 0
    },
    {
      "token": "中国人",
      "start_offset": 0,
      "end_offset": 3,
      "type": "CN_WORD",
      "position": 1
    },
    {
      "token": "中国",
      "start_offset": 0,
      "end_offset": 2,
      "type": "CN_WORD",
      "position": 2
    },
    {
      "token": "国人",
      "start_offset": 1,
      "end_offset": 3,
      "type": "CN_WORD",
      "position": 3
    },
    {
      "token": "人民共和国",
      "start_offset": 2,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 4
    },
    {
      "token": "人民",
      "start_offset": 2,
      "end_offset": 4,
      "type": "CN_WORD",
      "position": 5
    },
    {
      "token": "共和国",
      "start_offset": 4,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 6
    },
    {
      "token": "共和",
      "start_offset": 4,
      "end_offset": 6,
      "type": "CN_WORD",
      "position": 7
    },
    {
      "token": "国",
      "start_offset": 6,
      "end_offset": 7,
      "type": "CN_CHAR",
      "position": 8
    }
  ]
}
```

`ik_smart`的分词效果,尽可能少的返回了词语:

```json
{
  "tokens": [
    {
      "token": "中国人民",
      "start_offset": 0,
      "end_offset": 4,
      "type": "CN_WORD",
      "position": 0
    },
    {
      "token": "共和国",
      "start_offset": 4,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 1
    }
  ]
}
```

### 如何在查询中使用

要查询单个词,可以用match.查询多个词时,可以用terms,或者用空格分隔开.elasticsearch会返回包含它的文档.

```bash
get /testdatas/_search
{
    "query":{
        "match" : {
            "text" : "中国 人格"
        }
    }
}
```

查询分词器和索引分词器必须是同一个.如果分词没生效,检查一下是否是因为这个配置原因导致的.
{:.warning}


## 导入词库

有时候ik分词器自带的字典可能满足不了需求.通过导入自定义的词库可以扩充词典.

词典配置在`{conf}/analysis-ik/config/IKAnalyzer.cfg.xml` 或者 `{plugins}/elasticsearch-analysis-ik-*/config/IKAnalyzer.cfg.xml`下:
```xml
<!--用户可以在这里配置远程扩展字典 -->
<entry key="remote_ext_dict">location</entry>
<!--用户可以在这里配置远程扩展停止词字典-->
<entry key="remote_ext_stopwords">location</entry>
```
其中 location 是指一个 url，比如 `http://yoursite.com/getCustomDict`，该请求只需满足以下两点即可完成分词热更新。
 - 该 http 请求需要返回两个头部(header)，一个是 Last-Modified，一个是 ETag，这两者都是字符串类型，只要有一个发生变化，该插件就会去抓取新的分词进而更新词库。
 - 该 http 请求返回的内容格式是一行一个分词，换行符用 \n 即可。

满足上面两点要求就可以实现热更新分词了，不需要重启 ES 实例。

可以将需自动更新的热词放在一个 UTF-8 编码的 .txt 文件里，放在 nginx 或其他简易 http server 下，当 .txt 文件修改时，http server 会在客户端请求该文件时自动返回相应的 Last-Modified 和 ETag。可以另外做一个工具来从业务系统提取相关词汇，并更新这个 .txt 文件。




