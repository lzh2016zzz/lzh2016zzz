
---
title: 在redis中应尽量用scan代替keys命令
tags: 经验,redis
---

众所周知,redis是基于单线程的.

因此在使用一些时间复杂度 = O(n)的命令时要小心,可能一个不小心就会阻塞主进程,导致redis出现卡顿.

有时候,我们需要针对一部分符合条件的命令进行批量操作.比如获取以`ARTICLE:`开头的所有key.

那么要如何获得这些key呢?

在redis2.8之前,我们可以使用`KEYS`命令通过正则表达式获取我们想要的key.

但是`KEYS`命令有两个缺点:

1. 没有分页功能.我们只能一次性获取我们想要的key.如果结果数量有上千万上亿条,那么等待你的将会是"内存溢出".
2. `KEYS`是遍历算法,时间复杂度 = O(n),如果key的数量特别多,可能阻塞主线程导致redis出现卡顿.

所以,redis在2.8版本推出了`SCAN`命令.可以用于增量的迭代redis内的元素.

#### 关于Redis的SCAN命令

**SCAN cursor [MATCH pattern] [COUNT count]**

* cusor : 迭代时的下标.`SCAN`命令每次被调用以后,都会返回一个新的下标.把这个新的用于下次迭代以此延续之前的迭代过程.
* MATCH pattern : 用于匹配数据库key的正则表达式.
* count : 每次迭代返回的元素数量的最大值,也就是本次要遍历多少个元素,然后返回其中符合条件的值.

相比`KEYS`,`SCAN`有两个优点:

1. 提供了分页选项.可以控制每次迭代返回的元素数量.
2. 虽然`SCAN`的时间复杂度还是O(n),但是它是分批次进行的,不会阻塞线程.

正常情况下,使用`SCAN`不会有问题,但是如果正在rehash,就有可能出现重复数据.

另外,在迭代过程中被添加或者删除的元素有可能不会被返回.

这是由于`SCAN`的rehash机制导致的.

所以,在使用scan时,需要客户端进行去重.

#### 在spring中使用scan命令

有几个坑要注意:

1. cusor使用完以后需要关闭,否则会一直占用redis连接.这里使用了`try-with-resource`语法自动执行`AutoCloseable`的`close`方法.
2. 在执行时使用了`executeWithStickyConnection`,因为scan需要在同一个连接上执行.
3. scan返回的是`byte[]`,需要自己写`ConvertingCursor`进行反序列化.

```java
    class RedisScanExample{
    public void test() throws IOException {
        try (ConvertingCursor<byte[], String> cursor = getCursor()) {
            cursor.forEachRemaining(System.out::println);
        }
    }

    @SuppressWarnings("unchecked")
    private ConvertingCursor<byte[], String> getCursor() {
        return (ConvertingCursor<byte[], String>) redisTemplate
                .executeWithStickyConnection((RedisCallback<ConvertingCursor<byte[], String>>) connection -> {
            ScanOptions options = new ScanOptions
                    .ScanOptionsBuilder()
                    .match("test*")
                    .count(1000)
                    .build();
            return new ConvertingCursor<>(connection.scan(options), 
                    (b) -> (String) redisTemplate.getStringSerializer().deserialize(b));
        });
    }
}
```

额,虽然spring是事实标准但是我觉得这个api设计的有点过于底层了.

spring完全可以封装一个类似`scan(MATCHER,COUNT,TYPE,RESULT -> {})`的api让使用者直接调用.但是他没有这么干.

所以我建议在使用的时候封装一个类似`RedisUtil`的工具类供直接调用.
