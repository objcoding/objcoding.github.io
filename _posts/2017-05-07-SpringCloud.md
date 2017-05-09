---
layout: post
title:  "SpringCloud微服务分布式开发"
categories: BackEnd
tags: Spring SpringCloud 微服务 分布式
author: zch
---

> spring Cloud是一个基于**Spring Boot**实现的云应用开发工具，它为基于JVM的云应用开发中的配置管理、**服务发现**、**断路器**、**负载均衡**、**智能路由**、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。

现在公司的积分联盟平台系统构建于公司内部的第4代架构中，而第四代用得就是springCloud微服务架构，趁着项目上手，花了几天研究了一下。

### 1. 服务注册与发现---Netflix Eureka



### 2. 服务消费者---Netflix Ribbon
微服务架构就是将一个完整的应用从数据存储开始垂直拆分成多个不同的服务，每个服务都能独立部署、独立维护、独立扩展，服务与服务间通过诸如RESTful API的方式互相调用。而在公司项目中是通过spring框架提供RestTemplate来访问各个微服务提供的接口，而且还有负载均衡的作用。

举个例子：我要访问积分系统integral-server，添加积分用户：

```
JSONObject integralServerResult = restTemplate.postForObject("http://integral-server/shop/add", RequestHandler.getRestRawRequestEntity(integralShopJson), JSONObject.class);
```

配置bean：

```
@Bean
@LoadBalanced
RestTemplate restTemplate() {
	return new RestTemplate();
}
```


### 3. 断路器---Netflix Hystrix



### 4. 服务网关---Netflix Zuul



### 5. 分布式配置---Spring Cloud Config


