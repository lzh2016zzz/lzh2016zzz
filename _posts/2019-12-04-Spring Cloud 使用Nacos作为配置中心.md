
---
title: Spring Cloud 使用Nacos作为配置中心
tags: spring,nacos,spring cloud
---

## 什么是nacos

Nacos 是阿里巴巴开源的一个更易于构建云原生应用的注册中心、配置管理和服务管理平台

## 为什么需要配置中心

首先说说什么是配置中心.顾名思义,就是用来管理项目中所有模块的配置的系统.虽然听起来很简单.但是也不要小瞧了这个模块.如果一个中大型互联网项目,不使用配置中心的模式.一大堆的各类配置项，各种不定时的修改需求，一定会让开发非常头疼且管理十分混乱.

那么使用nacos作为配置中心有哪些好处呢?

* 在属性上使用`@NacosValue`注解,实现动态刷新配置,消除了配置变更时重新部署应用和服务的需要,让配置管理变得更加高效和敏捷.

* 配置支持版本控制.可以查看配置的历史记录(最多30天).方便追溯,还原对配置的修改.

* 支持灰度发布.Nacos 支持以IP为粒度的灰度配置.

* 更安全.配置跟随源代码保存在代码库中，容易造成配置泄露.配置保存在nacos中,可以防止配置泄露.


## 如何在spring-boot中使用nacos作为配置中心

#### 启动nacos


 * 通过github下载nacos  [nacos/releases](https://github.com/alibaba/nacos/releases)

* 下载解压后进入`bin`文件夹,运行命令`nohup sh startup.sh -m standalone &>/dev/null &`后台启动nacos

* 访问http://serverhost:8848/nacos/index.html 即可看到nacos管理页面,默认账号密码nacos/nacos

  

#### 配置namespace

通过namespace实现多环境下部署不同配置.

* 新建命名空间.进入`nacos管理页面` >`命名空间`> `新建命名空间` 填写命名空间名和描述"test" "测试环境配置"

  ![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/截屏2020-03-08下午4.31.57.png)

* 创建完成以后的列表页面

  ![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/截屏2020-03-08下午4.33.25.png)

#### 配置dataId

通过dataId为每一个服务创建一个配置,或者让多个服务使用同一套配置

* 点击`配置管理` > `配置列表` > `+`创建新配置

  ![](https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/截屏2020-03-08下午4.41.17.png)


* 点击`发布`创建配置成功

#### 通过maven配置引入相关依赖

添加依赖(这里本人使用的配置版本是 *0.2.1* ):

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>nacos-config-spring-boot-starter</artifactId>
    <version>${nacos-config-spring-boot.version}</version>
</dependency>
```

#### 开启配置

使用 `@NacosPropertySource` 加载 `dataId` 为 `pay-service` 的配置源，并开启自动更新

```java
@SpringBootApplication
@NacosPropertySource(dataId = "pay-service", autoRefreshed = true)
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```



#### 在工程中配置nacos地址和namespace

编辑`application-{环境}.yml`

```yml
nacos:
  config:
    server-addr: xx.xx.xxx.xxx:8848
    #这里填写的是命名空间id.每个环境一个命名空间,从而实现多环境配置.
    namespace: a27fad67-3fc0-4560-817a-b5f7db337656
```

#### 设置属性值

通过 Nacos 的 `@NacosValue` 注解向实体类注入属性值

```java
@Data
@Generated
@Component
public class ErpConfig {
    @NacosValue(value = "${pay.erp.contract.path.getContractByIdCard}", autoRefreshed = true)
    private String erpPathGetContractByIdCard;
}
```

使用的时候可以

```java
@Autowired
private ErpConfig erpConfig;

```

通过get方法获取值.

亦可直接在spring-bean内部使用`@NacosValue`直接设置属性值.



## nacos实现动态刷新配置的原理

Nacos 的客户端维护了一个长轮询的任务，去检查服务端的配置信息是否发生变更，如果发生了变更，那么客户端会拿到变更的 groupKey 再根据 groupKey 去获取配置项的最新值即可.
