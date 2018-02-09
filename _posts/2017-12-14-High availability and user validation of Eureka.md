---
layout: post
title: "Eureka的高可用与用户验证"
categories: SpringCloud
tags: Eureka
author: zch
---

* content
{:toc}
Eureka作为微服务的注册发现中心，在运作过程中保持稳定性尤为重要，如果一个项目只有一个 Eureka server，万一这个 Eureka server 宕机了，后果不堪设想，因此，我们需要搭建一个高可用的 Eureka server 集群；同样地为了安全考虑，还需要给 Eureka server 添加用户密码验证。









## 高可用

实现高可用的原理其实很简单，比如我们需要两个 Eureka server，就将这两个 Eureka server 相互注册到对方上，就可实现数据多个 Eureka server 直接的数据互相同步了。

- 互相注册：

```yaml

spring:
  application:
    name: eureka-master

---

spring:
  profiles: peer1
  
eureka:
  instance:
    preferIpAddress: true
    leaseRenewalIntervalInSeconds: 1
  client:
    register-with-eureka: false
    fetch-registry: false
    serviceUrl.defaultZone: http://zhangchenghui:123456@localhost:6002/eureka/

server:
  port: 6001

security:
  user:
    name: zhangchenghui
    password: 123456
        
---

spring:
  profiles: peer2
  
eureka:
  instance:
    preferIpAddress: true
    leaseRenewalIntervalInSeconds: 1
  client:
    register-with-eureka: false
    fetch-registry: false
    serviceUrl.defaultZone: http://zhangchenghui:123456@localhost:6001/eureka/

server:
  port: 6002

security:
  user:
    name: zhangchenghui
    password: 123456
```

配置文件可用 — 分割开，这样的好处就是可以将不同环境的配置写到一个配置文件中，运行的时候再指定要哪一段配置，比如上面如果在运行的时候 --spring.profiles.active=peer1，那么对应的那段就会应用到当前应用中。

将Eureka项目打包好，运行命令：

```bash
java -jar xxxxx.jar --spring.profiles.active=peer1
java -jar xxxxx.jar --spring.profiles.active=peer2
```



- 将应用注册到 Eureka server 集群上：

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://zhangchenghui:123456@localhost:6001/eureka/,http://zhangchenghui:123456@localhost:6002/eureka/
```

将这两个 eureka server 同时写到 defaultZone 中，这样就可以将服务注册到 Eureka server 集群上了。



## 用户验证



- 添加安全模块依赖：


```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```



- 将 Eureka 的 url 改成以下模式：


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



登陆 Eureka，看到以下弹窗，说明添加用户验证成功：

![Eureka](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/eureka.png)