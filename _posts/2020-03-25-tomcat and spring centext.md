---

layout: post
title: "在使用 SpringMVC 时，Spring 容器是如何与 Servlet 容器进行交互的？"
categories: Spring
tags: context servlet springmvc 容器
author: 张乘辉
---

* content
{:toc}
最近都在看小马哥的 Spring 视频教程，通过这个视频去系统梳理一下 Spring 的相关知识点，就在一个晚上，躺床上看着视频快睡着的时候，突然想到当我们在使用 SpringMVC 时，Spring 容器是如何与 Servlet 容器进行交互的？虽然在我的博客上还有几年前写的一些 SpringMVC 相关源码分析，但都是点到为止，没有好好再深入了解其中的机制，Spring 容器是如何与 Servlet 容器进行交互并没有交代清楚，于是趁着这个机会，再撸一次 SpringMVC 源码。






## Spring 容器的加载

可否还记得，当年还没有 Springboot 的时候，在 Tomcat 的 web.xml 中进行面向 xml 编程的青葱岁月？其中有那么几段配置总是令我记忆犹新：

首先是 Spring 容器配置：

```xml
<context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>classpath:spring-config.xml</param-value>
</context-param>
```

其次是 Servlet 容器监听器配置：

```xml
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

在 Tomcat 启动时，根据这两段配置，究竟做了什么动作，使得 Tomcat 与 Spring 完美地结合在一起了呢？

首先我们来看下 ContextLoaderListener 监听器的源码：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320205328.png)

我们发现它继承了 ContextLoader，并且实现了 ServletContextListener 接口，下面说下这两个东西的作用：

1. ContextLoader：正如其名，ContextLoader 可以在启动时载入 IOC 容器；
2. ServletContextListener：ServletContextListener 接口有两个抽象方法，contextInitialized 和 contextDestroyed，该监听器会结合 Web 容器的生命周期被调，ContextLoaderListener 正是实现了该接口。

因此，ContextLoaderListener 最主要的作用就是在 Tomcat 启动时，根据配置加载 Spring 容器。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320205759.png)

以上就是 ContextLoaderListener 实现 contextInitialized 方法的逻辑，也是加载并初始化 Spring 容器的开始。

org.springframework.web.context.ContextLoader#initWebApplicationContext

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320210832.png)

以上代码逻辑主要做了以下几个操作：

1. 调用 createWebApplicationContext 方法创建一个容器，会创建一个 contextClass 类型的容器，如果没有配置，则默认创建 WebApplicationContext 类型的容器；
2. 将容器强转为 ConfigurableWebApplicationContext 类型；
3. 调用 configureAndRefreshWebApplicationContext 方法初始化 Spring 容器；
4. 最后将 Spring 容器，以一个元素的形式保存到 Servlet 容器中，这也就意味着，得到 Servlet 容器，同时也可以得到 Spring 容器。

还发现 Spring 容器保存到 Servlet 容器中的 key 为 ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE，我们顺藤摸瓜找到获取 Spring 容器的方法：

org.springframework.web.context.support.WebApplicationContextUtils#getWebApplicationContext

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320220655.png)

关于这个方法在哪里调用后面有说到。

org.springframework.web.context.ContextLoader#configureAndRefreshWebApplicationContext

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320210900.png)

以上是 Spring 容器初始化逻辑，其中，CONFIG_LOCATION_PARAM 即是我们在 xml 中配置的 contextConfigLocation 参数：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320212506.png)

同时还会将 Servlet 容器保存到 Spring 容器中，最后调用 refresh 方法进行初始化。

在将 Spring 容器初始化最后以一个元素的形式保存到 Servlet 容器之后，那么 SpringMVC 在初始化时，是如何拿到 Spring 容器的呢？

我们继续看 SpringMVC 初始化是怎么操作的。



## SpringMVC 容器的加载

SpringMVC 本质上来讲，就是一个大号的 Servlet，其各种机制都是围绕着一个名叫 DispatcherServlet 的 Servlet 展开的，因此它必然实现了 Servlet 接口，那么在 Tomcat 启动时，它必然会通过 Servlet#init 方法进行初始化动作，我在其调用链路上发现以下方法：

org.springframework.web.servlet.FrameworkServlet#initWebApplicationContext

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320220032.png)

DispatcherServlet 的父类同样有一个方法，该方法是加载 SpringMVC 容器，即源码中的 webApplicationContext：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320221939.png)

我们发现，rootContext 就是 ContextLoaderListener 加载的 Spring 容器，在这里，它会以父容器的身份保存到 SpringMVC 容器中。

当然，如果是用 Springboot 环境，那么默认只会存在一个上下文环境，原因如下：

1、在 Springboot 应用程序启动时，在 SpringBootServletInitializer#onStartup 方法中，会创建一个 rootAppContext 容器，如下：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320235338.png)

**同时将上文所说的 ContextLoaderListener 监听器添加到 Servlet 容器中，同样达到了 xml 配置的效果**，而调用 createRootApplicationContext 方法创建 rootAppContext 容器时，会将 contextClass 设置为 AnnotationConfigServletWebServerApplicationContext.class。

2、DispatcherServlet 此时作为一个 Bean，实现了 ApplicationContextAware 接口，会自动将上下文环境保存到 webApplicationContext 字段中；

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320233807.png)

DispatcherServlet 初始化时，经过 debug 可以看到，rootContext 和 webApplicationContext 是同一个实例对象：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320231335.png)

原因是通过 ContextLoaderListener 加载的上下文环境，通过 ApplicationContextAware 接口自动 set 进来保存到 DispatcherServlet 的 webApplicationContext 变量中了。

在 FrameworkServlet#initWebApplicationContext 方法最后，最终会将 webApplicationContext 注入以一个元素的形式保存到 Servlet 容器中：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200320234137.png)



## DispatcherServlet 初始化

最终，SpringMVC 初始化会调用该方法：

org.springframework.web.servlet.DispatcherServlet#onRefresh

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200321000909.png)

DispatcherServlet 初始化时，从 Spring 容器中获取相关 Bean，初始化各种不同的组件，比如初始化 HandlerMapping：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200321001725.png)



## 总结

本质上来讲，Servlet 容器与 Spring 容器并不互通，但因为有 Servlet 容器的监听器 ServletContextListener，在它们之间构筑了桥梁。