---
title:学习Elasticsearch的利器: kibana
tags: 
  elasticsearch
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/cover4.jpg
---



在学习Elasticsearch的过程中,必须要有一些工具用来编写查询,以及查看elasticsearch的数据.kibana可以完美的帮助你学习和使用es

<!--more-->

# 安装

通过docker安装kibana,kibana的版本号要和elasticsearch一致:

```bash
docker run -d --name mykibana \
      -e ELASTICSEARCH_URL=http://{你的elasticsearch ip:端口号} \
      -p 5601:5601 \
      kibana:6.8.2
```

安装完以后通过`localhost:5601`就可以进行访问了:

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/截屏2021-07-01 22.44.45.png)

# 使用

这个页面可以看到左侧的导航菜单.重点说一下比较常用的工具:

##  Dev Tools

 `console`可以协助我们的方便地编写查询请求,**支持自动补全**

 `Search Profiler`帮助我们分析执行计划,找到慢查询的原因

`Grok Debugger` 用于在线运行调试Grok表达式

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/截屏2021-07-01 22.51.48.png)

## Monitoring

这个页面可以方便的查看集群的状态:

- Overview : 集群的整体使用情况.每秒写入数,查询数,延迟
- Nodes : 集群内部每个节点的使用情况
- Indices : 集群内部索引的使用情况 

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/截屏2021-07-01 22.58.44.png)

## Management

管理索引.包括新增删除冻结生命周期等.

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/截屏2021-07-01 23.02.15.png)