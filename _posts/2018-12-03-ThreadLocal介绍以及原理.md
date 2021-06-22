
---
title: ThreadLocal介绍以及原理
tags: java,技术
---

### 先看 <<JAVA并发编程实战>>中对`ThreadLocal`的定义:

* ThreadLocal是一个关于创建线程局部变量的类。用于维持**线程封闭**的变量。

  

###### 我的理解:

​	通常情况下,我们创建的变量是可以被任何一个线程访问并修改的.

​	而使用`ThreadLocal`创建的变量只能被当前线程访问，其他线程则无法访问和修改.

​	因为`ThreadLocal`只能被当前线程访问,所以不存在多线程之间的共享问题.


### 那么,什么情况下需要使用`ThreadLocal`呢?

###### 避免参数传递

举个例子.对于java web应用而言,Session保存了很多的用户信息.很多时候需要通过Session获取和修改应用信息.

一方面，需要保证每个线程有自己单独的 Session 实例。另一方面，由于很多地方都需要操作 Session，存在多方法共享 Session 的需求。

如果不使用 ThreadLocal，可以在每个线程内构建一个 Session实例，并将该实例在多个方法间传递.但是每个需要使用 Session 的地方，都需要显式传递 Session 对象，方法间耦合度较高。

使用 ThreadLocal 的代码，不再需要在各个方法间传递 Session 对象，并且也非常轻松的保证了每个线程拥有自己独立的实例.

###### 设置线程上下文

最典型的,就是用于管理数据库的连接.每个线程对应一个连接,当前线程的所有数据库操作都是交给同一个连接管理.从而实现了事务管理.Hibernate对事务的管理就是基于`ThreadLocal`实现的.



做一下总结:

* `ThreadLocal` 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，也即变量在线程间隔离而在方法或类间共享的场景。

* 另外，该场景下，并非必须使用 ThreadLocal ，其它方式完全可以实现同样的效果，只是 ThreadLocal 使得实现更简洁。



### ThreadLocal的实现原理

想要更好的理解`ThreadLocal`,那么就有必要了解它的实现原理.

因为同一个`ThreadLocal`包裹的对象在不同的线程中有不同的副本,所以`ThreadLocal`内部其实是维护了一个map.

我们通过`get()`,`set()`方法设置threadLocal的值时,实际上是在调用这个map对应的`get()`,`set()`方法.

代码如下:

###### set方法

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

###### get方法

```java
 public T get() {   
        Thread t = Thread.currentThread();   
        ThreadLocalMap map = getMap(t);   
        if (map != null)   
            return (T)map.get(this);      
    // Maps are constructed lazily.  if the map for this thread   
    // doesn't exist, create it, with this ThreadLocal and its   
    // initial value as its only entry.   
    T value = initialValue();   
    createMap(t, value);   
    return value;   
}
```
###### ThreadLocalMap是个静态的内部类：

```java
    static class ThreadLocalMap {   
    ........   
    }  
```

也就是说,通过`ThreadLocal`存取的变量其实是放在线程共享的`ThreadLocalMap`中.`ThreadLocal`只是对`ThreadLocalMap`做了一层封装.



### ThreadLocal在线程池中可能引起内存泄露问题

感谢`sonarlint`弥补了我的知识盲区.

正常情况下使用`ThreadLocal`不会有问题,因为JVM利用设置ThreadLocalMap的Key为弱引用，来避免内存泄露.

为什么弱引用可以避免内存泄露?

弱引用的特点是，如果这个对象只存在弱引用，那么在下一次垃圾回收的时候必然会被清理掉.

假设 引用`ThreadLocal`变量的线程被回收了,这个变量在在ThreadLocalMap的对应的Key因为只剩下弱引用,所以会被垃圾回收掉.从而避免了泄露问题.

而线程池复用了线程资源,延长了线程的生命,那么就有可能增加内存泄露的风险. 

###### 如何避免问题?

在线程池中 使用完 `ThreadLocal`变量以后.手动将其设置为null.即可避免线程泄露问题.


