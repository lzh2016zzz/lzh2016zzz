---
title:  spring bean 详解
tags: spring,读书笔记
---


## 什么是spring bean

Spring bean是由spring容器管理的对象.Spring 容器会自动完成bean对象的实例化.

这也是IOC的本质,由容器来管理对象的生命周期.

## bean的实例化过程&&生命周期

bean的实例化过程如下图:

![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/20200226165417.png)

Bean的完整生命周期经历了各种方法调用，这些方法可以划分为以下几类：

1. **Bean自身的方法**

   包括了Bean内部带有`@PreDestroy`和`@PostConstruct` 和通过配置文件中<bean>的init-method和destroy-method指定的方法

2. **Bean级生命周期接口方法**

   包括了`BeanNameAware`、`BeanFactoryAware`、`InitializingBean`和`DiposableBean`这些接口的方法

3. **容器级生命周期接口方法**

   包括了`InstantiationAwareBeanPostProcessor` 和` BeanPostProcessor` 这两个接口实现，一般称它们的实现类为“后处理器”。

4. **工厂后处理器接口方法**

   包括了`AspectJWeavingEnabler`,` ConfigurationClassPostProcessor`, CustomAutowireConfigurer等等非常有用的工厂后处理器接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用。

## bean的作用域和实现方式

1. **singleton**
    单例  默认的作用域
    
    ##### 实现方式 
    在获取bean之前,会尝试从bean缓存中获取bean,如果存在,则直接返回对应的bean,否则会先通过beandefinition实例化bean并缓存.(该作用域适用于无状态的bean)
    优点 : 一次创建成功以后可以重复使用.是spring bean默认的作用域.
2. **prototype**
   
    ##### 实现方式 
    每次获取bean时,都会创建一个新的实例.
    
    适用于有状态的bean
3. **request & session**
    顾名思义,这类bean 只在当前的request | session 内有效.
    
    ##### 实现方式
    
    * 通过RequestContextListener  监听 request的创建,通过Threadlocal保证每个线程都持有对应的request
    * 实例化bean 
    * 把实例化的bean放入request 域中
    该作用域只支持在web容器中使用spring时创建
 4. **global session**
    每个全局的HTTP Session，使用session定义的Bean都将产生一个新实例。典型情况下，仅在使用portlet context的时候有效。同样只有在Web应用中使用Spring时，该作用域才有效



## 常见的依赖注入方式 

常用的依赖注入方式有三种::

* setter注入

  在setter方法上使用`@Autowired`或者`@Resource`注解.Spring会自动查找并注入对应的bean.

* 构造器注入(spring官方推荐的注入方式)

  在Bean的构造器上添加需要的依赖作为参数,Spring会自动查找并注入对应的bean

  构造器注入的优点 :

  * 保证依赖不可变（final关键字）

  * 保证依赖不为空（省去了我们对其检查）

  * 保证返回客户端（调用）的代码的时候是完全初始化的状态

  * 避免了循环依赖(通过启动时抛出异常的方式进行)

  * 提升了代码的可复用性

* 属性注入

  在属性上使用`@Autowired`或者`@Resource`注解.Spring会自动查找并注入对应的bean.



虽然官方推荐使用构造器注入.

但是个人认为,面向service编程本质上是面向表结构的面向过程编程.

所以绝大多数情况下,使用setter注入和属性注入并无不良影响.还能省去处理循环依赖异常的麻烦.


