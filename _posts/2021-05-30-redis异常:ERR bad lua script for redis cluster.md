---
title: redis异常:ERR bad lua script for redis cluster
tags: 经验,redis,速查
---

当我使用redissonClient.getMapCache(),执行put操作时,会抛出一个异常:

*ERR bad lua script for redis cluster, all the keys that the script uses should be passed using the KEYS array, and KEYS should not be in expression. channel: [id: 0xe15ab90b, L:/xx.xx.xxx:38542 - R:xxx.redis.rds.aliyuncs.com/xx.xx.xxxx:6379*

这个问题只能在阿里云redis上重现.内网redis集群运行正常.

## 解决方法

修改阿里云redis中script_check_enable参数为0即可解决该问题

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/92126769-26ce0480-ee33-11ea-8130-cd7ea23ac79e.png)

## 原因

默认阿里云Redis会对lua脚本做一定限制，目的是为了保证脚本里面的所有操作都在相同slot进行，如果用户能够在代码确保所有操作都在相同slot而又想打破Redis集群的lua限制的话可以通过控制台修改script_check_enable参数为0，则后端不会对脚本进行校验。

云Redis集群对lua脚本限制如下：

- 所有key都应该由 KEYS 数组来传递，redis.call/pcall 中调用的redis命令，key的位置必须是KEYS array（不能使用Lua变量替换KEYS），否则直接返回错误信息，"-ERR bad lua script for redis cluster, all the keys that the script uses should be passed using the KEYS arrayrn"。
- 所有key必须在1个slot上，否则返回错误信息，"-ERR eval/evalsha command keys must be in same slotrn"。
- 调用必须要带有key，否则直接返回错误信息， "-ERR for redis cluster, eval/evalsha number of keys can't be negative or zerorn"。

参考:

[https://developer.aliyun.com/article/645851](https://developer.aliyun.com/article/645851)

