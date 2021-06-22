
---
title: Spring Cloud Open Feign详解
tags: spring,spring cloud
---

## Fegin介绍

Feign是一个声明式的http客户端,只需要创建一个接口并加上注解,就可以通过feigin调用http接口. 另外Feign默认集成了Ribbon和Hystrix，并和Eureka结合，默认实现了负载均衡和熔断器的效果。

简单来说:

* Feign使用基于接口的注解进行定义
* Feign整合了Ribbon和Hystrix
* 使用Feign可以像调用原生java方法一样调用其他微服务暴露的接口


## 如何通过Feign调用其它微服务

##### 需要在maven工程引入依赖

```xml
<parent>
      <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.0.RELEASE</version>
        <relativePath/>
</parent>


<dependencies>
<!-- feign -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
</dependencies>
```


##### 在启动类上加上`@EnableFeignClients`注解以启用Feign客户端

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class ServiceFeignApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceFeignApplication.class, args);
    }
}
```

##### 定义一个Fegin接口,通过`@FeignClient("服务名")`选择调用的服务,通过定义方法指定需要定义的接口

```java
@FeignClient(value = "bas")
public interface BasRemoteProxy {

    @PostMapping("/api/business-flow")
    BusinessDTO pushBasWater(@RequestBody BusinessDTO businessDTO);

    //使用query参数 
    //需要在参数前面加上注解 @RequestParam("参数名称")
    @PostMapping("/sms/sendMessage")
    void sendMessage(@RequestParam("smsSign") String smsSign, @RequestParam("phone") String phone, @RequestParam("smsContent") String smsContent);

    //使用url占位符
    //需要在参数前面加上注解 @PathVariable("参数名称")
    @GetMapping("/lending-organizations/{organizationCode}")
    FundProductDTO getLendingOrganization(@PathVariable(value = "organizationCode") Long organizationCode);

    //使用对象作为get参数
    //需要使用 @SpringQueryMap 注解
    @GetMapping(path = "/data-access/all")
    Page<DataAccess> dataAccessQuery(@SpringQueryMap PageableParams params);
}
```



--------------------



## 在Feign中使用熔断器(Hystrix)进行服务降级处理

##### Hystrix熔断器的作用

用一句话概括,就是在调用服务失败/出现异常时,提供一个服务降级的处理逻辑.

对于查询操作, 我们可以实现一个fallback方法, 当请求后端服务出现异常的时候, 可以使用fallback方法返回的值. fallback方法的返回值一般是设置的默认值或者来自缓存.



##### 触发Hystrix熔断器的条件

* 接口调用超时.

* 接口调用抛出了非`HystrixBadRequestException`的异常

* 并发数量超过了Hystrix线程池的线程数目.那么多余的请求会被抛弃.

* 有大量请求到达,而且有多于一定百分比的请求失败.

  

如何使用 ?

##### 通过`application.yml` 配置启用Hystrix

```yml
feign:
  hystrix:
    enabled: true
```



接下来我们要给Feign 客户端`BasRemoteProxy`配置fallback.配置fallback有两种方式:

* 实现Feign接口

* 实现`FallbackFactory`

  

###### 方式1:Fallback实现 - 实现Feign接口

通过实现`BasRemoteProxy`接口实现,需要通过`@Component`注册为spring组件

`BasRemoteProxyFallBack`类:

```java
//
@Component("basRemoteProxyFallBack")
@Slf4j
public class BasRemoteProxyFallBack implements BasRemoteProxy {

    @Override
    public BusinessDTO pushBasWater(BusinessDTO businessDTO) {
        log.error("调用pushBasWater接口失败");
        //模拟内部调用失败
        throw new RuntimeException("内部异常");
    }

    @Override
    public void sendMessage(String smsSign, String phone, String smsContent) {
        log.error("调用sendMessage接口失败");
    }

    @Override
    public FundProductDTO getLendingOrganization(Long organizationCode) {

        log.error("调用getLendingOrganization接口失败");
        return null;
    }
}
```

`BasRemoteProxy`配置:

```java
@FeignClient(value = "bas", fallback = BasRemoteProxyFallBack.class)
public interface BasRemoteProxy {

......
```

###### 方式2 : Fallback实现 - 实现FallbackFactory

上面的方法实现方式简单，但是获取不到HTTP请求错误状态码和信息 ，这时就可以使用工厂模式来实现`Fallback`,同样需要通过`@Component`注册为spring组件

`BasRemoteProxyFallBackFactory`类:

```java
@Component
@Slf4j
public class BasRemoteProxyFallBackFactory implements FallbackFactory<BasRemoteProxy> {

    @Override
    public BasRemoteProxy create(Throwable throwable) {

        //打印异常信息
        log.error("BasRemoteProxy调用异常",throwable);
        //编写fallback,为了直观,
        return new BasRemoteProxy() {

            @Override
            public BusinessDTO pushBasWater(BusinessDTO businessDTO) {
                log.error("调用pushBasWater接口失败");
                //模拟内部调用失败
                throw new RuntimeException("内部异常");
            }

            @Override
            public void sendMessage(String smsSign, String phone, String smsContent) {
                log.error("调用sendMessage接口失败");
            }

            @Override
            public FundProductDTO getLendingOrganization(Long organizationCode) {

                log.error("调用getLendingOrganization接口失败");
                return null;
            }
        };
    }
}
```

`BasRemoteProxy`配置:

```java
@FeignClient(value = "bas", fallbackFactory = BasRemoteProxyFallBackFactory.class)
public interface BasRemoteProxy {
```



##### 设置超时时间

```yml
feign:
  client:
      config:
          default:
                  #连接超时时间
              connectTimeout: 100000
              # 处理超时时间
              readTimeout: 1000000
```



### 为Feign自定义 Error Decoder

在使用Feign调用微服务,触发熔断机制时,会出现一些问题:

* 调用失败时,异常信息不直观.类似：`BasRemoteProxy#pushBasWater(ParamVO) failed and no fallback available.`

* 无法获取到服务提供者抛出的原始异常信息

通过自定义 `ErrorDecoder`可以解决上面的问题.

##### 如何编写

当调用服务时，如果服务返回的`http status >= 300`且不是`404`，就会进入到`Feign`的`ErrorDecoder`中，因此如果���们要解析异常信息，就要实现`ErrorDecoder`接口.

`ErrorDecoder`接口:

```java
public interface ErrorDecoder {
  public Exception decode(String methodKey, Response response);
}
```

参数: 

* methodKey :

   出现异常的FeginClient类#方法名(参数) 

   例:  `BasRemoteProxy#pushBasWater(ParamVO)`

* response: 

  封装了从客户端接收到的响应值.

  
  

编写`FeignErrorDecoder`类:

  ```java
@Configuration
@ConditionalOnClass(ErrorDecoder.class)
public class FeignErrorDecoder extends ErrorDecoder.Default implements ErrorDecoder {

    private static final Logger log = LoggerFactory.getLogger(FeignErrorDecoder.class);

    @Override
    public Exception decode(String methodKey, Response response) {
        String message = StringUtils.EMPTY;
        if (response.body() != null) {
            try {
                message = Util.toString(response.body().asReader());
            } catch (IOException e) {
                log.error("feign 异常处理失败：", e);
            }
        }
        // 这里直接拿到我们抛出的异常信息
        try {
            JsonNode jsonNode = JsonUtils.readTree(message);
            //获取errorCode
            int errorCode = jsonNode.get("errorCode").asInt(NumberUtils.INTEGER_MINUS_ONE);
            //获取异常信息message
            String errorMessage = jsonNode.get("message").asText(StringUtils.EMPTY);
            //根据code和message包装不同类型的异常
            switch (errorCode) {
                case 417:
                    return new BizException(errorCode, errorMessage);
                case 500:
                    if(errorMessage.contains("NullPointerException")){
                        return new NullPointerException(errorMessage);
                    }
                    if(errorMessage.contains("IllegalArgumentException")){
                        return new IllegalArgumentException(errorMessage);
                    }
                    if(errorMessage.contains("IllegalStateException")){
                        return new IllegalStateException(errorMessage);
                    }
                default:
                    return new UnhandledFeignException(errorCode, message);
            }
        } catch (Exception e) {
            log.error("feign 异常处理失败：", e);
            String msg = format("status %s reading %s", response.status(), methodKey);
            if (response.body() != null) {
                msg += "; content:\n" + message;
            }
            return new DecodeException(msg);
        }
    }
}
  ```

  

