---
layout: post
title: "SpringMVC初始化过程"
categories: JavaSE/EE
tags: Spring MVC
author: zch
---

* content
{:toc}

Servlet创建时的初始化过程中，会调用init()方法，我们可以在init()方法内做一系列的初始化动作，比如给Servlet的私有变量赋值，事实上SpringMVC在初始化过程中经常干这种事情。我们知道，在Servlet初始化过程中，Servlet把init-param的属性放在ServletConfig里，然后在循环它拿到值，但是这样太费劲了，所以SpringMVC索性把init-param赋值到自己的私有属性中。

HttpservletBean继承了HttpServlet，所以它也是一个Servlet，它也是FrameworkServlet的父类，而FrameworkServlet是DispatcherServlet的父类，每个类都有自己相关的初始化动作，这是典型的Template Method模板设计模式，每个类都会完成一些自己的本职工作，把不属于自己的本职工作推给自己的子类去完成。





## HttpservletBean初始化

上面我们也提到HttpservletBean继承了HttpServlet，所以SpringMVC的初始化工作就从HttpservletBean的init()方法开始：

```java
@Override
public final void init() throws ServletException {
  if (logger.isDebugEnabled()) {
    logger.debug("Initializing servlet '" + getServletName() + "'");
  }

  // Set bean properties from init parameters.
  try {
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
    ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
    bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
    initBeanWrapper(bw);
    bw.setPropertyValues(pvs, true);
  }
  catch (BeansException ex) {
    logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
    throw ex;
  }

  // 这个细节留给子类实现
  initServletBean();

  if (logger.isDebugEnabled()) {
    logger.debug("Servlet '" + getServletName() + "' configured successfully");
  }
}
```

```java
protected void initServletBean() throws ServletException {}
```

HttpServletBean 初始化过程全在 init 方法中定义，它的核心就是创建 **BeanWrapper**。

通过 BeanWrapper 可以访问到 Servlet 的所有参数、资源加载器加载的资源以及 DispatcherServlet 中的所有属性，并且可以像访问 Javabean 一样简单。

## FrameworkServlet初始化

```java
@Override
protected final void initServletBean() throws ServletException {
  // 此处省略部分代码
  try {
    this.webApplicationContext = initWebApplicationContext();
    initFrameworkServlet();
  }
  // 此处省略部分代码
}
```

```java
protected WebApplicationContext initWebApplicationContext() {
  // 取得根容器
  WebApplicationContext rootContext =
    WebApplicationContextUtils.getWebApplicationContext(getServletContext());
  WebApplicationContext wac = null;

  // 取得SpringMVC容器
  if (this.webApplicationContext != null) {
    // A context instance was injected at construction time -> use it
    wac = this.webApplicationContext;
    if (wac instanceof ConfigurableWebApplicationContext) {
      ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
      if (!cwac.isActive()) {
        // The context has not yet been refreshed -> provide services such as
        // setting the parent context, setting the application context id, etc
        if (cwac.getParent() == null) {
          // The context instance was injected without an explicit parent -> set
          // the root application context (if any; may be null) as the parent
          cwac.setParent(rootContext);
        }
        configureAndRefreshWebApplicationContext(cwac);
      }
    }
  }
  // 此处省略部分代码
  // 重刷新 SpringMVC 容器
  if (!this.refreshEventReceived) {
    // Either the context is not a ConfigurableApplicationContext with refresh
    // support or the context injected at construction time had already been
    // refreshed -> trigger initial onRefresh manually here.

    // 交给子类实现
    onRefresh(wac);
  }
  // 此处省略部分代码

  // 初始化容器
  wac.setEnvironment(getEnvironment());
  wac.setParent(parent);
  wac.setConfigLocation(getContextConfigLocation());
  configureAndRefreshWebApplicationContext(wac);
  return wac;
}
```

FrameworkServlet 在 SpringMVC 的初始化的过程中主要负责其容器的初始化。它取得 Spring 根容器的目的是将其设为 SpringMVC 的父容器。

## DispatcherServlet初始化

```java
@Override
protected void onRefresh(ApplicationContext context) {
  initStrategies(context);
}
```

```java
protected void initStrategies(ApplicationContext context) {
  // 初始化多文件上传解析器
  initMultipartResolver(context);
  // 初始化区域解析器
  initLocaleResolver(context);
  // 初始化主题解析器
  initThemeResolver(context);
  // 初始化处理器映射集
  initHandlerMappings(context);
  // 初始化处理器适配器
  initHandlerAdapters(context);
  // 初始化处理器异常解析器
  initHandlerExceptionResolvers(context);
  // 初始化请求到视图名转换器
  initRequestToViewNameTranslator(context);
  // 初始化视图解析器
  initViewResolvers(context);
  // 初始化 flash 映射管理器
  initFlashMapManager(context);
}
```

DispatcherServlet 初始化的工作就是负责初始化各种不同的组件。它好比给 SpringMVC 添加功能模块，一旦功能模块添加完毕，SpringMVC 就可以正常工作。因此 SpringMVC 的初始化工作也到此结束。
