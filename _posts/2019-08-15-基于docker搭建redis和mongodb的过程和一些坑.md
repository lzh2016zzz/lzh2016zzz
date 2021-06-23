---
title: 基于docker搭建redis和mongodb的过程和一些坑
tags: 
   - docker
   - redis
   - mongo
---


我是一个docker小白.这段时间正在学习如何使用docker.

因为测试环境需要,所以想通过docker搭建redis和mongodb.记录一下搭建过程留作以后备用.



<!--more-->

## docker搭建redis

```bash
#单机模式,不带密码
docker run --name redis -p 127.0.0.1:6379:6379 -v /etc/localtime:/etc/localtime -d redis
#单机模式,带密码
docker run --name redis -p 127.0.0.1:6379:6379 -v /etc/localtime:/etc/localtime -d redis --requirepass "密码"
```


## docker搭建mongodb

```bash
#单机,无认证
docker run -p 127.0.0.1:27017:27017 -v <LocalDirectoryPath>:/data/db --name docker_mongodb -d mongo
#单机,带身份验证的
docker run -d --name mongodb  -v /home/xxx/dbdir/mongodb:/data/db -e MONGO_INITDB_ROOT_USERNAME=xxx -e  MONGO_INITDB_ROOT_PASSWORD=xxxxx -p 127.0.0.1:27017:27017  mongo --auth
#-p 指定容器的端口映射，mongodb 默认端口为 27017

#-v 为设置容器的挂载目录，这里是将<LocalDirectoryPath>即本机中的目录挂载到容器中的/data/db中作为 mongodb 的存储目录

#--name 为设置该容器的名称

#-d 设置容器以守护进程方式运行

#注意 : 数据迁移的时候只需要把<LocalDirectoryPath>下的文件挪走就好了
```

## 搭建过程中的一些坑

* 报错 : SELinux is not supported with the overlay2 graph driver on this kernel

  根据百度的结果, 出现这个问题是因为此linux的内核中的SELinux不支持 overlay2 graph driver ，解决方法有两个，要么启动一个新内核，要么就在docker里禁用selinux.

```bash
#编辑/etc/sysconfig/docker

$ vi /etc/sysconfig/docker

#找到这行配置

OPTIONS='--selinux-enabled=false  --log-driver=journald --signature-verification=false'

#删除 '--selinux-enabled=false'  

#删除完以后大概是这样

OPTIONS='--log-driver=journald --signature-verification=false'

#重启docker
$ systemctl restart docker
```

* 报错:RUN sudo mkdir -p ... cannot allocate memory

  查了一下文档,发现这个错误的意思就像它自己说的一样,无法分配内存了.

  解决方案(暂时的):

  ```bash
  #写入pid_max
  echo "kernel.pid_max=999999" >> /etc/sysctl.conf
  #使配置重新生效
  sysctl -p
  ```

  后来我发现这个问题是有原因的.
  具体表现是从某次更新以后服务的线程数就不断增加,从最初的几百,一千多,慢慢增长到一万,两万..初步判断是将系统最大线程占满导致的..
  导致系统最大线程占满的原因很多,这次是因为中了门罗币的挖矿病毒.
  当然实际情况多变,配置异常,中毒,代码中有创建大量线程的逻辑,都有可能.

* 报错:Error response from daemon: error creating overlay mount to ... invalid argument.
  
  出现这个错误是因为当前的内核不支持overlay2文件系统.
  解决方案:
  
  ```bash
  #step1：先停用docker服务
  $ service docker stop
  #step2：删除docker镜像文件夹
  $ rm -rf /var/lib/docker
  #step3：重新指定docker文件系统
  $ vi /etc/sysconfig/docker-storage
   # 找到下面的参数，做如下修改：
   DOCKER_STORAGE_OPTIONS="--storage-driver overlay "
   # 保存
   $ !wq
  #step4：重启docker服务
  $ service docker start
  ```

  


