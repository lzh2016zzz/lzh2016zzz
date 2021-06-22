---
title: spring-boot配置分布式对象网格
tags: 
   - spring
   - redis
---


## Redisson官方介绍

Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。其中包括(`BitSet`, `Set`, `Multimap`, `SortedSet`, `Map`, `List`, `Queue`, `BlockingQueue`, `Deque`, `BlockingDeque`, `Semaphore`, `Lock`, `AtomicLong`, `CountDownLatch`, `Publish / Subscribe`, `Bloom filter`, `Remote service`, `Spring cache`, `Executor service`, `Live Object service`, `Scheduler service`) Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

## 配置

pom.xml:

```xml
<!-- version:3.12.1 -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>${redisson.version}</version>
<dependency>
```


application.yml:

```yml
redis:
  host: redis-host
  port: 6379
  password: meiyoumima
  jedis:
    pool:
      max-idle: 10
      min-idle: 0
      max-active: 16
      max-wait: -1
```

RedisConfig.java:

```java
/**
 * @Author: lzh
 * @Date: 2020/2/2 1:21 下午
 */
@Configuration
public class RedisConfig {


    @Bean
    public RedissonClient redissonClient(RedisProperties redisProperties) {
        Config config = new Config();
        SingleServerConfig singleServerConfig = config.useSingleServer().setAddress("redis://" + redisProperties.getHost() + ":" + redisProperties.getPort());
        if (StringUtils.isNotBlank(redisProperties.getPassword())) {
            singleServerConfig.setPassword(redisProperties.getPassword());
        }
        RedissonClient redissonClient = Redisson.create(config);
        return redissonClient;
    }

}
```

## 使用

```java
//注入bean
@Autowired
private RedissonClient redissonClient;
		
   /**
     * 一键获取各种分布式的java常用对象和服务
     */
    public void test(){
        //获取读写锁
        ReadWriteLock readWriteLock = redissonClient.getReadWriteLock("LOCK_NAME");
        //获取独占锁
        Lock lock = redissonClient.getLock("LOCK_NAME");
        //获取map
        Map<Object, Object> map = redissonClient.getMap("MAP");
        //获取布隆过滤器
        RBloomFilter<Object> bloomFilter =redissonClient.getBloomFilter("BLOOM_FILTER");
    }
```
