---
layout: post
title: "SpringCloud微服务架构之服务注册与发现"
categories: SpringCloud
tags: Eureka
author: 张乘辉
---

* content
{:toc}
现在公司的积分联盟平台系统构建于公司内部的第4代架构中，而第四代用得就是springCloud微服务架构，趁着项目上手，花了几天研究了一下。

SpringCloud是一个庞大的分布式系统，它包含了众多模块，其中主要有：服务发现（Eureka），断路器（Hystrix），智能路有（Zuul），客户端负载均衡（Ribbon）等。也就是说微服务架构就是将一个完整的应用从数据存储开始垂直拆分成多个不同的服务，每个服务都能独立部署、独立维护、独立扩展，服务与服务间通过诸如RESTful API的方式互相调用。











## 创建服务注册中心

- 在搭建SpringCloud分布式系统前我们需要创建一个注册服务中心，已便监控其余模块的状况。这里需要在pom.xml中引入：


```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

- 并且在SpringBoot主程序中加入@EnableEurekaServer注解：


```java
@EnableEurekaServer
@SpringCloudApplication
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

- 接下来在SpringBoot的属性配置文件application.properties中如下配置：


```properties
server.port=9100
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:${server.port}/eureka/
```

server.port就是你指定注册服务中心的端口号，在启动服务后，可以通过访问http://localhost:9100服务发现页面，如下：

![注册服务中心](https://gitee.com/objcoding/md-picture/raw/master/img/springcloud.png)





## 创建服务方

我们可以发现其它系统在这里注册并显示在页面上了，想要注册到服务中心，需要在系统上做一些配置，步骤跟创建服务注册中心类似，这里web-gateway系统做例子：

- 首先在pom.xml中加入：


```xml
 <dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-eureka</artifactId>
 </dependency>
```

- 在SpringBoot主程序中加入@EnableDiscoveryClient注解，该注解能激活Eureka中的`DiscoveryClient`实现，才能实现Controller中对服务信息的输出：


```java
@EnableDiscoveryClient
@SpringBootApplication
public class WebGatewayApplication {
	public static void main(String[] args) {
		SpringApplication.run(WebGatewayApplication.class, args);
	}
}
```

- 在SpringBoot的属性配置文件application.properties中如下配置：


```properties
spring.application.name=web-gateway
server.port=9010
eureka.client.serviceUrl.defaultZone=http://localhost:9100/eureka/
eureka.instance.leaseRenewalIntervalInSeconds=5
```

再次启动服务中心，打开链接：http://localhost:9100/，就可以看到刚刚创建得服务了。
