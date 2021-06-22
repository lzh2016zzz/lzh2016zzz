
---
title: Spring Cloud 选择合适的注册中心
tags: spring,spring cloud
---

## 什么是注册中心
在微服务架构中,注册中心是最核心的基础服务之一.在微服务架构流行之前,注册中心就已经开始出现在分布式架构的系统中.

注册中心可以说是微服务架构中的”通讯录“，它记录了服务和服务地址的映射关系.在分布式架构中,服务会注册到这里,当服务需要调用其它服务时,就到这里找到服务的地址,进行调用.



#### 举个例子

* 没有注册中心的情况

  假设我们知道微服务的IP和端口号,那么我们就可以调用它.

  但是如果微服务换了地址或者端口,或者增加了新的节点,我们该如何得到通知? 

* 有注册中心的情况

  调用微服务时,通过服务名查找到对应的微服务的地址,然后就可以直接调用它.

  如果微服务出现异动或者新增节点,那么微服务也会告知注册中心.

  后续,我们调用微服务时,就可以获取到更新以后的地址.



上述两个场景就是注册中心的两个职责：

* 服务发现

* 服务注册

## Nacos和Eureka

简单介绍一下Nacos和Eureka,目前Spring Cloud体系中最常用的两个注册中心.

* Eureka是Netflix开发的一个基于rest的服务发现框架.SpringCloud将它集成在其子项目spring-cloud-netflix中，以实现SpringCloud的服务发现功能.


* Nacos 是阿里巴巴开源的一个更易于构建云原生应用的注册中心、配置管理和服务管理平台.同时具有服务发现 和 配置中心的功能.

首先,两者部署方式有差别.eureka注册中心需要启动一个Spring Boot工程.然后通过引入依赖 + 自动装配的方式启用 + 部署.nacos则是通过官网下载jar包然后直接启动.

其次,对于客户端来说,两者差别不大,都是通过注解开启服务发现功能. 

另外,虽然nacos server也是基于spring boot实现的,但是其自带的后台管理界面无疑对使用者来说更加友好.而且nacos还提供另一个核心功能 - 动态配置中心.

相比eureka,nacos在功能性方面无疑是有优势的.

最关键的.eureka2.x闭源了.继续使用它需要承担风险.而nacos有阿里维护.



## 从Eureka无痛迁移到Nacos

使用 Spring Cloud Alibaba 的开源组件 `spring-cloud-starter-alibaba-nacos-discovery`替换 Eureka，So Easy！

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-alibaba-nacos-discovery -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-alibaba-nacos-discovery</artifactId>
    <version>0.2.1.RELEASE</version>
</dependency>
```

##### application配置
修改application配置`application.properties`

```properties
nacos.discovery.server-addr=127.0.0.1:8848
#有人说不生效，就试下下面的
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
```

##### 如果是application.yml，请试下下面的配置

```yml
nacos:
    discovery:
      server-addr: 127.0.0.1:8848
#建议先用上面的
spring:
    cloud:
      nacos:
        discovery:
          server-addr: 127.0.0.1:8848
```

##### @EnableDiscoveryClient注解

更换EnableEurekaClient 注解。如果在你的应用启动程序启动类加了`@EnableEurekaClient` ，请修改为`@EnableDiscoveryClient` 

##### 其它的需要注意的点

如何不停止服务的前提下更换注册中心? 有没有合适的方案呢?

