---
layout: post
title:  "SpringCloud微服务架构之服务消费者"
categories: SpringCloud
tags: Ribbon
author: 张乘辉
---

* content
{:toc}
在系统与系统之间，如何进行相互间得调用呢？也就是说怎么去调用服务提供得接口内容呢？这里就要说一下Ribbon了，Ribbon是一个基于http和tcp客户端得负载均衡器，什么是负载均衡呢？简单来说就是一个就是一个域名地址设置多个地址，当我们访问这个域名地址时，定位到服务器分发中心，再根据各个服务器的负载状况进行分发，达到均衡状态，试想一下如果全部访问都集中到一台服务器，那不得直接崩了？











下面我来简单介绍如何在SpringCloud分布式系统下使用Ribbon来实现负载均衡。

- 首先在pom.xml中引入一下依赖：


```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

- 然后在spring boot主程序中创建RestTemplate类，并为它加上@LoadBalanced注解开启负载均衡的能力：


```java
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
}
```

RestTemplate类是Spring用于构建Restful服务而提供的一种Rest服务可客户端，RestTemplate 提供了多种便捷访问远程Http服务的方法，能够大大提高客户端的编写效率，所以很多客户端比如 Android或者第三方服务商都是使用 RestTemplate 请求 restful 服务。

- 在apllication.properties配置文件中配置eureka服务，并注册到服务中心：


```properties
spring.application.name=integral-server
server.port=9600
eureka.client.serviceUrl.defaultZone=http://localhost:9100/eureka/
```

- 在公司项目中正是通过RestTemplate来访问各个微服务提供的接口，比如在项目中要访问积分系统integral-server，添加积分用户：


```java
JSONObject integralServerResult = restTemplate.postForObject("http://integral-server/shop/add", RequestHandler.getRestRawRequestEntity(integralShopJson), JSONObject.class);
```
这样就可以调用integral-server系统的添加用户的接口实现在别的系统中添加用户了。

- 我们也可以在application.properties配置文件中加入：


```properties
###### Ribbon
ribbon.ReadTimeout=60000
```

这个是设置负载均衡的超时时间的。
