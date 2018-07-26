---
layout: post
title: "SpringCloud微服务架构之feign客户端"
categories: SpringCloud
tags: feign http
author: zch
---

* content
{:toc}
之前说过了微服务间，我是通过 Spring 的 RestTemplate 类来相互调用的，它可通过整合 Ribbon 实现负载均衡，但发现了这样写不够优雅，且不够模板化，因此本篇介绍一下 Feign。

Feign是一种声明式、模板化的 HTTP 客户端，在 Spring Cloud 中使用 Feign 其实就是创建一个接口类，它跟普通接口没啥两让，因此通过 Feign 调用 HTTP 请求，开发者完全感知不到这是远程方法。









## 整合 Feign

- 添加 Feign 依赖：

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

- 创建 一个 Feign 接口：

```java
@FeignClient(value = FeignConst.COUPON_PROVIDER, url = "${feign.coupon.url:}")
public interface CouponClient {

  @GetMapping(value = "/coupon/list/page", headers = LocalsEncoder.CONTENT_TYPE_LOCALS_GET)
  RestResponse couponList(@ModelAttribute CouponCriteria criteria);

}
```

- 启动 Feign 类

```java
@EnableFeignClients(basePackages = {"com.objcoding"})
@SpringCloudApplication
public class ProviderApplication {
	
}
```





## 服务降级

当网络不稳定时，一个接口响应非常慢，就会一直占用这个连接资源，如果长时间不做处理，会导致系统雪崩，幸好，Feign 已经继承了熔断器 Hystrix

```java
@FeignClient(value = FeignConst.COUPON_PROVIDER, url = "${feign.coupon.url:}", fallback = CouponClient.CouponClientFallBack.class)
public interface CouponClient {

  @GetMapping(value = "/coupon/list/page", headers = LocalsEncoder.CONTENT_TYPE_LOCALS_GET)
  RestResponse couponList(@ModelAttribute CouponCriteria criteria);

  @Component
  class CouponClientFallBack implements CouponClient {
    @Override
    public RestResponse couponList(CouponCriteria criteria) {
      return RestResponse.failed("网络超时");
    }
  }

}
```





## 拦截器

有时候微服务间的调用，需要传递权限信息，这些信息都包含在请求头了，这时我们可以通过 Feign 拦截器实现权限穿透：

```java
@Configuration
public class WebRequestInterceptor {

  @Bean
  public RequestInterceptor headerInterceptor() {
    return template -> {
      ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
      if (attributes == null) {
        return;
      }
      HttpServletRequest request = attributes.getRequest();
      Enumeration<String> headerNames = request.getHeaderNames();
      if (headerNames != null) {
        while (headerNames.hasMoreElements()) {
          String name = headerNames.nextElement();
          String values = request.getHeader(name);
          template.header(name, values);
        }
      }
    };
  }
}
```







