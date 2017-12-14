---
layout: post
title: "Eureka的高可用与用户验证"
categories: SpringCloud
tags: Eureka
author: zch
---

* content
{:toc}




## 高可用Eureka





## 用户验证





```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```





```yaml
eureka:
  instance:
    preferIpAddress: true
    leaseRenewalIntervalInSeconds: 1
  client:
	serviceUrl.defaultZone: http://userName:password@eureka-master:6001/eureka/
```







```yaml
security:
  user:
    name: zhangchenghui
    password: xxxxx
```

