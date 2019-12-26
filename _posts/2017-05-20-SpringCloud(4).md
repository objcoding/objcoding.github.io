---
layout: post
title:  "SpringCloud微服务架构之服务网关"
categories: SpringCloud
tags: Zuul
author: zch
---

* content
{:toc}
到这里我们就可以构建一个略微不足的微服务分布式系统了，前面我们通过Ribbon实现服务的消费和负载均衡，但还有些不足的地方，举个例子，服务A和服务B，他们都注册到服务注册中心，这里还有个对外提供的一个服务，这个服务通过负载均衡提供调用服务A和服务B的方法，但是这样做就破坏了服务无状态的特点，无法直接复用既有接口，当我们需要对一个即有的集群内访问接口，实现外部服务访问时，我们不得不通过在原有接口上增加校验逻辑，或增加一个代理调用来实现权限控制，无法直接复用原有的接口。

最好的方法就是把所有请求都集中在最前端的地方，这地方就是zuul服务网关。

服务网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供REST API的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制等功能。Spring Cloud Netflix中的Zuul就担任了这样的一个角色，为微服务架构提供了前门保护的作用，同时将权限控制这些较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。













## 配置服务路由

- 要使用zuul，就要引入它的依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```

- 在spring boot主程序中加入@EnableZuulProxy注解开启zuul：

```java
@EnableEurekaClient
@EnableZuulProxy
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

- 在application.properties配置文件中配置zuul路由url：

```properties
spring.application.name=web-gateway
server.port=9010
```

到这里，一个微服务zuul服务网关系统已经可以运行了，接下来就是如何配置访问其它微服务系统的url，zuul提供了两种配置方式，一种是通过url直接映射，另一种是利用注册到eureka server中的服务id作映射：

- url直接映射：

```properties
zuul.routes.api-a-url.path=/api-a-url/**
zuul.routes.api-a-url.url=http://localhost:2222/
```

所有到Zuul的中规则为：`/api-a-url/**`的访问都映射到`http://localhost:2222/`上，比如 ：`http://localhost:9010/api-a-url/add?a=1&b=2`，该请求会转发到：`http://localhost:2222/add?a=1&b=2`上。

- 但是这么做必须得知道所有得微服务得地址，才能完成配置，这时我们可以利用注册到eureka server中的服务id作映射：

```properties
###### Zuul配置
zuul.routes.api-integral.path=/integral/**
zuul.routes.api-integral.serviceId=integral-server

zuul.routes.api-member.path=/member/**
zuul.routes.api-member.serviceId=member-server
```

integral-server和member-server是这俩微服务系统注册到微服务中心得一个serverId，我们通过配置，访问`http://localhost:9010/integual/add?a=1&b=2`，该请求就会访问integral-server系统中得add服务。



## 服务过滤

在配置完服务路由之后, 我们还需要配置服务顾虑，来限制客户端只访问到指定访问的资源，提高系统的安全性。

在定义zuul网关服务过滤只需要创建一个继承ZuulFilter抽象类并重写四个方法即可：

- filterType：返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型，具体如下：

  pre：可以在请求被路由之前调用；

  routing：在路由请求的时候被调用；

  post：在routing和error过滤器之后被调用；

  error：在请求发生错误的时候被调用

- filterOrder：通过int值来定义过滤器的执行顺序

- shouldFilter：返回一个boolean类型来判断该过滤器是否要执行，所以通过此函数可实现过滤器的开关。在上例中，我们直接返回true，所以该过滤器总是生效。

- run：过滤器的具体逻辑。需要注意，这里我们通过`ctx.setSendZuulResponse(false)`令zuul过滤该请求，不对其进行路由，然后通过`ctx.setResponseStatusCode(401)`设置了其返回的错误码，当然我们也可以进一步优化我们的返回，比如，通过`ctx.setResponseBody(body)`对返回body内容进行编辑等。

实例程序：

```java
public class ErrFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        Object accessToken = request.getParameter("accessToken");
        if(accessToken == null) {
            log.warn("access token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            return null;
        }
        return null;
    }
}
```

在自定过滤器之后，我们还需要在SpringBoot主程序中加入@EnableZuulProxy注解来开启zuul路由的服务过滤：

```java
@EnableZuulProxy
@EnableEurekaClient
@RibbonClients
@SpringCloudApplication
public class ApiGatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(ApiGatewayApplication.class, args);
	}

	@Bean
	PosPreFilter posPreFilter(){
		return new PosPreFilter();
	}
```

到这里，微服务系统的zuul路由功能基本搭建完成，如下图（来源于网上）：

![zuul-filter](https://gitee.com/objcoding/md-picture/raw/master/img/zuul-filter.png)