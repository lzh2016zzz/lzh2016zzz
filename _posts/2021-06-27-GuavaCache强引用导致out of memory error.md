---
tags: 
   - java
   - bug排查
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/cover3.jpg
---


生产环境的某个集群偶尔出现`out of memory error`.

VM监控表现: 服务器负载不高的情况下,老年代内存短时间内直线上涨,多次触发FGC直到OOM.这里记录一下问题的原因和解决方案,方便以后查阅.


<!--more-->


# 排查问题

## 配置

服务器启动时要加上配置,这样在OOM时就能将内存快照记录下来:

```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${目录}
```

## 通过VisualVM分析内存快照

VisualVM是Java自带的监控工具.有诸多强大的功能,其中一个很重要的功能是用于分析JVM内存快照:

### 查看占用内存最多的对象

* 打开hprof文件 

* 选择`Objects` 

*  `Retained` (倒序)

排第一的是`com.google.common.cache.LocalCache$LocalManualCache`,占用内存高达3.5GB.这个类属于Guava Cache,一个用于实现本地缓存的库

### 排查代码

通过`Reference`定位到超大对象所属的类`GuavaCacheManager`,下面的代码是它的构造器.

`listeners`用于监听key的新增/删除操作,并通过Zookeeper中间件同步给集群内部的其他机器.

```java
	public GuavaCacheManager(String name, long expire, List<CacheListener> listeners) {
		this.listeners = listeners;
		this.name = name;
		this.expire = expire;
		CacheBuilder<Object, Object> build = CacheBuilder.newBuilder().maximumSize(maxSize);
		if (expire != 0) {
			build.expireAfterAccess(expire, TimeUnit.SECONDS);
		}
		//监听器代码,略过
		cache = build.build();
	}
```

这行代码构建了一个Guava缓存,它接收一个size参数,当缓存内的key数量大于size时,则会触发淘汰策略(LRU),删除使用最少的key:

```java
CacheBuilder<Object, Object> build = CacheBuilder.newBuilder().maximumSize(maxSize);
```

经过排查,size设置的太大是导致问题的直接原因,这导致key的数量还没来得及增长到触发对象回收就已经报内存不足了.

把淘汰阈值改小并不能解决问题.首先很难确定一个够用的阈值,阈值设置的太小会导致频繁触发淘汰,设置的太大那么跟没设置一样.

解决方法来自之前了解的`memcached`原理.使用软引用就行了.

### Java的引用类型

- 强引用: 最常见的引用.如果一个对象具有强引用，那垃圾回收器绝不会回收它.

- 软引用: 在即将触发OOM时,GC会尝试回收这类对象.之后如果内存还不够则抛出OOM异常.

- 弱引用: 每次触发GC时都会被回收.

- 虚引用: 对对象的生命周期没有影响.但可以在对象被回收时得到一个通知.


默认情况下,Guava Cache使用**强引用**存储Key和Value.也支持软引用.将缓存的key/value改为软引用,问题就解决了:

```java
CacheBuilder<Object, Object> build = CacheBuilder.newBuilder().softValues().softValues().maximumSize(maxSize);
```

所以说多读些书,多了解些原理还是很有好处的.这样遇到一些没有头绪的问题的时候就能触类旁通.


复习一下本地缓存和分布式缓存的区别:

### 本地缓存和分布式缓存

本地缓存访问速度快,相对于分布式缓存省去了/IO/序列化/反序列化开销,但是由于缓存数据存放在内存中,所以无法进行大数据存储. 

* 适用于数据量较小但是访问频次高的数据

分布式缓存由独立的集群支撑,支持大数据存储,具备高可用,高性能,以及读写分离等优秀特性

* 适用于绝大部分场景

# 参考资料

[https://guava.dev/releases/snapshot/api/docs/com/google/common/cache/CacheBuilder.html](https://guava.dev/releases/snapshot/api/docs/com/google/common/cache/CacheBuilder.html)