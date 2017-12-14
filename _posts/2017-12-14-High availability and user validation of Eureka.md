---
layout: post
title: "Eureka的高可用与用户验证"
categories: SpringCloud
tags: Eureka
author: zch
---

* content
{:toc}
Eureka作为微服务的注册发现中心，在运作过程中保持稳定性尤为重要，如果一个项目只有一个Eureka server，万一这个Eureka server宕机了，后果不堪设想，因此，我们需要搭建一个高可用的Eureka server集群；同样地为了安全考虑，还需要给Eureka server添加用户密码验证。









## 高可用Eureka









## 用户验证



- 添加安全模块依赖：


```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```



- 将Eureka的url改成以下模式：


```yaml
eureka:
  instance:
    preferIpAddress: true
    leaseRenewalIntervalInSeconds: 1
  client:
	serviceUrl.defaultZone: http://userName:password@eureka-master:6001/eureka/
```



- 在配置文件添加如下配置：


```yaml
security:
  user:
    name: zhangchenghui
    password: xxxxx
```



登陆eureka，看到以下弹窗，说明添加用户验证成功：

![Eureka](https://raw.githubusercontent.com/zhangchenghuidev/zhangchenghuidev.github.io/master/images/eureka.png)