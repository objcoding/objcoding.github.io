---
layout: post
title:  "SpringCloud微服务分布式系统的开发"
categories: BackEnd
tags: Spring SpringCloud 微服务 分布式
author: zch
---

> spring Cloud是一个基于**Spring Boot**实现的云应用开发工具，它为基于JVM的云应用开发中的配置管理、**服务发现**、**断路器**、**负载均衡**、**智能路由**、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。

现在公司的积分联盟平台系统构建于公司内部的第4代架构中，而第四代用得就是springCloud微服务架构，趁着项目上手，花了几天研究了一下。

SpringCloud是一个庞大的分布式系统，它包含了众多模块，其中主要有：服务发现（Eureka），断路器（Hystrix），智能路有（Zuul），客户端负载均衡（Ribbon）等。也就是说微服务架构就是将一个完整的应用从数据存储开始垂直拆分成多个不同的服务，每个服务都能独立部署、独立维护、独立扩展，服务与服务间通过诸如RESTful API的方式互相调用。

### 1. 服务注册与发现---Netflix Eureka

#### 1.1 创建服务注册中心

在搭建SpringCloud分布式系统前我们需要创建一个注册服务中心，已便监控其余模块的状况。这里需要在pom.xml中引入：

```
<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

并且在SpringBoot主程序中加入@EnableEurekaServer注解

```
@EnableEurekaServer
@SpringCloudApplication
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

接下来在SpringBoot的属性配置文件application.properties中如下配置：

```
server.port=9100
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:${server.port}/eureka/
```

server.port就是你指定注册服务中心的端口号，在启动服务后，可以通过访问http://localhost:9100服务发现页面，如下：

![注册服务中心](https://raw.githubusercontent.com/zchdjb/zchdjb.github.io/master/images/springcloud.png)

#### 1.2 创建服务方

我们可以发现其它系统在这里注册并显示在页面上了，想要注册到服务中心，需要在系统上做一些配置，步骤跟创建服务注册中心类似，这里web-gateway系统做例子：

首先在pom.xml中加入：

```
 <dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-eureka</artifactId>
 </dependency>
```

在SpringBoot主程序中加入@EnableDiscoveryClient注解，该注解能激活Eureka中的`DiscoveryClient`实现，才能实现Controller中对服务信息的输出。

```
@EnableDiscoveryClient
@SpringBootApplication
public class WebGatewayApplication {
	public static void main(String[] args) {
		SpringApplication.run(WebGatewayApplication.class, args);
	}
}
```

在SpringBoot的属性配置文件application.properties中如下配置：

```
spring.application.name=web-gateway
server.port=9010
eureka.client.serviceUrl.defaultZone=http://localhost:9100/eureka/
eureka.instance.leaseRenewalIntervalInSeconds=5
```

再次启动服务中心，打开链接：http://localhost:9100/，就可以看到刚刚创建得服务了

### 2. 服务消费者---Netflix Ribbon

那么，在系统与系统之间，如何进行相互间得调用呢？也就是说怎么去调用服务提供得接口内容呢？这里就要说一下Ribbon了，Ribbon是一个基于http和tcp客户端得负载均衡器，什么是负载均衡呢？简单来说就是一个就是一个域名地址设置多个地址，当我们访问这个域名地址时，定位到服务器分发中心，再根据各个服务器的负载状况进行分发，达到均衡状态，试想一下如果全部访问都集中到一台服务器，那不得直接崩了？

下面我来简单介绍如何在SpringCloud分布式系统下使用Ribbon来实现负载均衡。

首先在pom.xml中引入一下依赖：

```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

然后在spring boot主程序中创建RestTemplate类，并为它加上@LoadBalanced注解开启负载均衡的能力：

```
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

在公司项目中正是通过RestTemplate来访问各个微服务提供的接口，比如在项目中要访问积分系统integral-server，添加积分用户：

```
JSONObject integralServerResult = restTemplate.postForObject("http://integral-server/shop/add", RequestHandler.getRestRawRequestEntity(integralShopJson), JSONObject.class);
```
这样就可以调用integral-server系统的添加用户的接口实现在别的系统中添加用户了。

我们也可以在application.properties配置文件中加入：

```
###### Ribbon
ribbon.ReadTimeout=60000
```

这个是设置负载均衡的超时时间的。

### 3. 断路器---Netflix Hystrix

 在整个微服务系统架构中，各个系统被拆分成一个个服务单元，个单元间又通过诸如负载均衡的方式互相依赖，而每个服务单位都在不同的服务器中运行，假设其中一个服务器崩了又或者是网络延迟，这也就会导致其它依赖此服务的服务单元b也会出现延迟，而依赖此服务单元b的服务单元c也会出现延迟，如此一来，整个微服务系统就会出现雪崩效应，导致整个系统瘫痪。

为了解决此问题，须在系统中加入断路器模式，“断路器”本身就是一个很形象的词，因为断路器模式的作用也类似于电路中的断路器，一旦发生短路，断路器就会切断电源（熔断保险丝），确保电路安全。而微服务系统中，一旦某个服务单元发生瘫痪后者网络延迟等原因，断路器模式监控到此异常，就会向调用方返回一个错误响应，而不是让调用方长时间在等待，这样就不会产生在一个线程中被长时间占用，提高了系统的响应速度。

在SpringCloud个模块中有一个叫Netflix Hystrix的断路器模块。Hystrix是Netflix开源的微服务框架套件之一，该框架目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备拥有回退机制和断路器功能的线程和信号隔离，请求缓存和请求打包，以及监控和配置等功能。

下面来说一下Hystrix在微服务系统中的具体用法：

首先还是在pom.xml中加入以下依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

在spring boot主程序中加入@EnableCircuitBreaker注解开启断路器模式：

```
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

我们也可以在application.properties配置文件中加入：

```
## hystrix
hystrix.commond.default.execution.isolation.thread.timeoutInMilliseconds=60000
```

这个设置可以更改返回错误响应的超时时间。

如果不想返回默认的错误响应信息，我们还可以通过自定义来更改错误响应信息，我们需要一个类中注入一个RestTemplate类：

```
 @Autowired
 RestTemplate restTemplate;
```

这个类在上面已经通过Spring创建好了，这里直接注入在类中即可，接下来我们在类中写一个方法：

```
@HystrixCommand(fallbackMethod = "addServiceFallback")
    public String addService() {
        return restTemplate.postForObject("http://integral-server/shop/add", RequestHandler.getRestRawRequestEntity(integralShopJson), JSONObject.class);
    }
    public String addServiceFallback() {
        return "error";
    }
```

当调用integral-server系统的添加接口超出延时的时间时，就会返回“error”。

### 4. 服务网关---Netflix Zuul



### 5. 分布式配置---Spring Cloud Config


