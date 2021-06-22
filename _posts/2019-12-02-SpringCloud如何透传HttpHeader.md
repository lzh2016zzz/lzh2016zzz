---
title: SpringCloud如何透传HttpHeader
tags: 
   - spring
   - spring cloud
---


## 什么是透传

其实很简单,引用<<维基百科>>中的定义:

> 透传，即透明传输（pass-through），指的是在通讯中不管传输的业务内容如何，只负责将传输的内容由源地址传输到目的地址，而不对业务数据内容做任何改变。


## 如何在微服务之间透传HttpHeader

两种方式:

* 通过在请求上,或者在类上添加`@Headers`注解

* 通过实现`RequestInterceptor`接口,对特定的请求设置Header


##### 通过添加注解的方式

```java
@Headers({"Content-Type: application/json","Outer-Key: {OuterKey}"})
@PostMapping(value = "/card-blank/batch-create")
Response batchCreateCard(@RequestBody CreateCardBlankDTO condition,@Param("OuterKey") String type);
```

*  使用 Header : {参数名称} 可以传递动态header属性



##### 2.使用`RequestInterceptor`

在`@FeginClient`上引入配置类`configuration = FeignConfig.class`,使用配置类的好处是可以复用配置.

```java
@FeignClient(value = "bas", configuration = FeignConfig.class)
public interface BasRemoteProxy {
```

`FeignConfig.class`类:

```java
@Configuration
public class FeignConfig {


    @Bean
    @Primary
    public RequestInterceptor requestInterceptor() {
        return new HeaderInterceptor();
    }

    /**
     * 保证Hystrix熔断时仍可以传递信息
     */
    @Bean
    @Qualifier("contractfeignHystrixConcurrencyStrategy")
    public FeignHystrixConcurrencyStrategy contractfeignHystrixConcurrencyStrategy() {
        return new FeignHystrixConcurrencyStrategy();
    }

    public static class HeaderInterceptor implements RequestInterceptor {

        /**
         * 服务间需要透传的头
         */
        private final Set<String> HEADER_NAMES = new HashSet<>(Arrays.asList("tenant_code", "oauth2-authentication", "oauth2-authority"));

        public HeaderInterceptor(String ...args) {
            HEADER_NAMES.addAll(Arrays.asList(args));
        }

        @Override
        public void apply(RequestTemplate requestTemplate) {

            // 微服务透传header
            ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder
                    .getRequestAttributes();
            if (null == attributes || null == attributes.getRequest()) {
                return;
            }
            HttpServletRequest request = attributes.getRequest();
            Enumeration<String> headerNames = request.getHeaderNames();
            if (headerNames != null) {
                while (headerNames.hasMoreElements()) {
                    String headerName = headerNames.nextElement();
                    if (HEADER_NAMES.contains(headerName)) {
                        String value = request.getHeader(headerName);
                        requestTemplate.header(headerName, value);
                    }
                }
            }

        }
    }
}
```
