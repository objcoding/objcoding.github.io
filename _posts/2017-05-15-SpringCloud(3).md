---
layout: post
title:  "SpringCloud微服务架构之断路器"
categories: SpringCloud
tags: Hystrix
author: zch
---

* content
{:toc}
 在整个微服务系统架构中，各个系统被拆分成一个个服务单元，个单元间又通过诸如负载均衡的方式互相依赖，而每个服务单位都在不同的服务器中运行，假设其中一个服务器崩了又或者是网络延迟，这也就会导致其它依赖此服务的服务单元b也会出现延迟，而依赖此服务单元b的服务单元c也会出现延迟，如此一来，整个微服务系统就会出现雪崩效应，导致整个系统瘫痪。

为了解决此问题，须在系统中加入断路器模式，“断路器”本身就是一个很形象的词，因为断路器模式的作用也类似于电路中的断路器，一旦发生短路，断路器就会切断电源（熔断保险丝），确保电路安全。而微服务系统中，一旦某个服务单元发生瘫痪后者网络延迟等原因，断路器模式监控到此异常，就会向调用方返回一个错误响应，而不是让调用方长时间在等待，这样就不会产生在一个线程中被长时间占用，提高了系统的响应速度。

在SpringCloud个模块中有一个叫Netflix Hystrix的断路器模块。Hystrix是Netflix开源的微服务框架套件之一，该框架目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备拥有回退机制和断路器功能的线程和信号隔离，请求缓存和请求打包，以及监控和配置等功能。















下面来说一下Hystrix在微服务系统中的具体用法：

- 首先还是在pom.xml中加入以下依赖：


```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

- 在spring boot主程序中加入@EnableCircuitBreaker注解开启断路器模式：


```java
@EnableEurekaClient
@EnableCircuitBreaker
@EnableDiscoveryClient
@SpringBootApplication
public class WebGatewayApplication {

	@Bean
	@LoadBalanced
	RestTemplate restTemplate() {
		return new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(WebGatewayApplication.class, args);
	}
```

如果在调用过程中返回类似这样的响应：

```
Whitelabel Error Page

This application has no explicit mapping for /error, so you are seeing this as a fallback.

Sat May 13 00:10:22 CST 2017
There was an unexpected error (type=Internal Server Error, status=500).
400 null
```

断路器也就开启了。

- 我们也可以在application.properties配置文件中加入：


```properties
## hystrix
hystrix.commond.default.execution.isolation.thread.timeoutInMilliseconds=60000
```

这个设置可以更改返回错误响应的超时时间。

- 如果不想返回默认的错误响应信息，我们还可以通过自定义来更改错误响应信息，我们需要一个类中注入一个RestTemplate类：


```java
 @Autowired
 RestTemplate restTemplate;
```

这个类在上面已经通过Spring创建好了，这里直接注入在类中即可，接下来我们在类中写一个方法：

```java
@HystrixCommand(fallbackMethod = "addServiceFallback")
    public String addService() {
        return restTemplate.postForObject("http://integral-server/shop/add", RequestHandler.getRestRawRequestEntity(integralShopJson), JSONObject.class);
    }
    public String addServiceFallback() {
        return "error";
    }
```

当调用integral-server系统的添加接口超出延时的时间时，就会返回“error”。
