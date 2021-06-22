---
title: Elasticsearch安装
tags: elasticsearch
---


安装的版本是elasticsearch6.7.2,在安装之前需要先确认JDK版本大于1.8

## 检查jdk版本

```bash
java -version
echo $JAVA_HOME
```

查看JDK的版本，正确安装JDK后，就可以下载安装ES了

## 下载文件

在[https://www.elastic.co/cn/downloads/past-releases/elasticsearch-6-7-2](https://www.elastic.co/cn/downloads/past-releases/elasticsearch-6-7-2)下载安装es6.7版本

## 启动

在linux/macOS下执行:

```bash
cd elasticsearch/bin
./elasticsearch
```

在windows下执行:

```bash
./elasticsearch.bat
```

用浏览器访问localhost:9200,可以看到如下的信息:
```json
{
  "name" : "gXASjB0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "xFNBNVl8S2KhAOlmV0FnMw",
  "version" : {
    "number" : "6.7.2",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "8453f77",
    "build_date" : "2019-03-21T15:32:29.844721Z",
    "build_snapshot" : false,
    "lucene_version" : "7.7.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

观察返回的信息可以得到如下信息:

1. 集群名称是"elasticsearch",版本号是6.7.2,lucene_version = "7.7.0"

## 使用docker搭建elasticsearch集群

### 安装docker

macOS/Windows可以在官网下载docker-desktop安装包: [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)

macOS可以通过homebrew命令行安装docker:

```bash
brew install homebrew/cask/docker
```

### 检查docker服务是否启动

```bash
docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
# 需要手动启动服务
CONTAINER ID   IMAGE                                    COMMAND                  CREATED       STATUS                PORTS
# 启动成功
```

docker服务启动成功以后,就可以开始安装elasticsearch了

安装elasticsearch,并将容器9200端口映射到宿主机上:

```bash
docker run -d --name elasticsearch  \
          elasticsearch:6.7.2 \
          -p 127.0.0.1:9200:9200/tcp
```


