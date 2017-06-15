---
layout: post
title: "SpringMVC工作原理"
categories: Java
tags: Spring Controller MVC
author: zch
---

* content
{:toc}
## 1. 初始化

Servlet创建时的初始化过程中，会调用init()方法，我们可以在init()方法内做一系列的初始化动作，比如给Servlet的私有变量赋值，事实上SpringMVC在初始化过程中经常干这种事情。我们知道，在Servlet初始化过程中，Servlet把init-param的属性放在ServletConfig里，然后在循环它拿到值，但是这样太费劲了，所以SpringMVC索性把init-param赋值到自己的私有属性中。

HttpservletBean继承了HttpServlet，所以它也是一个Servlet，它也是FrameworkServlet的父类，而FrameworkServlet是DispatcherServlet的父类，每个类都有自己相关的初始化动作，这是典型的Template Method模板设计模式，每个类都会完成一些自己的本职工作，把不属于自己的本职工作推给自己的子类去完成。

### 1.1 HttpservletBean初始化

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

### 1.2 FrameworkServlet初始化

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

### 1.3 DispatcherServlet初始化

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

## 2. 请求处理流程

前面也说过，HttpServletBean继承了Httpservlet，因此它也遵循Servlet流程来处理用户请求，其主要有doGte()和doPost()，分别处理用户的GET请求和POST。

### 2.1 FrameworkServlet请求处理

```java
@Override
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
  throws ServletException, IOException {
  processRequest(request, response);
}

@Override
protected final void doPost(HttpServletRequest request, HttpServletResponse response)
  throws ServletException, IOException {
  processRequest(request, response);
}
```

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
  throws ServletException, IOException {

  long startTime = System.currentTimeMillis();
  Throwable failureCause = null;
  // 取得当前线程的 LocaleContext 。默认为空
  LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
  // 创建LocaleContext，该接口表示区域内容。主要作用是封装请求的 Locale 信息
  LocaleContext localeContext = buildLocaleContext(request);
  // 取得当前线程的 RequestAttributes 。默认为空
  RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
  // 2.创建 RequestAttributes，该接口封装了本次请求的 rqeust，response 信息。通过它可以很方便的操作 rqeust 的属性 
  ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

  WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
  asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
  // 3.初始化 ContextHolders
  initContextHolders(request, localeContext, requestAttributes);

  try {
    // 服务处理，留给子类实现
    doService(request, response);
  }
  catch (ServletException ex) {
    failureCause = ex;
    throw ex;
  }
  catch (IOException ex) {
    failureCause = ex;
    throw ex;
  }
  catch (Throwable ex) {
    failureCause = ex;
    throw new NestedServletException("Request processing failed", ex);
  }

  finally {
    // 5. 释放 ContextHolders
    resetContextHolders(request, previousLocaleContext, previousAttributes);
    if (requestAttributes != null) {
      requestAttributes.requestCompleted();
    }

    if (logger.isDebugEnabled()) {
      if (failureCause != null) {
        this.logger.debug("Could not complete request", failureCause);
      }
      else {
        if (asyncManager.isConcurrentHandlingStarted()) {
          logger.debug("Leaving response open for concurrent processing");
        }
        else {
          this.logger.debug("Successfully completed request");
        }
      }
    }

    publishRequestHandledEvent(request, response, startTime, failureCause);
  }
}
```

```java
protected abstract void doService(HttpServletRequest request, HttpServletResponse response)throws Exception;
```

可以看出，FrameworkServlet把GET请求和POST请求都统一用processRequest()来处理，而具体的服务细节交给子类DispatcherServlet去处理。

### 2.1.2 DispatcherServlet请求处理

```java
@Override
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
  if (logger.isDebugEnabled()) {
    String resumed = WebAsyncUtils.getAsyncManager(request).hasConcurrentResult() ? " resumed" : "";
    logger.debug("DispatcherServlet with name '" + getServletName() + "'" + resumed +
                 " processing " + request.getMethod() + " request for [" + getRequestUri(request) + "]");
  }

  // Keep a snapshot of the request attributes in case of an include,
  // to be able to restore the original attributes after the include.
  Map<String, Object> attributesSnapshot = null;
  if (WebUtils.isIncludeRequest(request)) {
    attributesSnapshot = new HashMap<String, Object>();
    Enumeration<?> attrNames = request.getAttributeNames();
    while (attrNames.hasMoreElements()) {
      String attrName = (String) attrNames.nextElement();
      if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
        attributesSnapshot.put(attrName, request.getAttribute(attrName));
      }
    }
  }

  // Make framework objects available to handlers and view objects.
  request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
  request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
  request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
  request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

  FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
  if (inputFlashMap != null) {
    request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
  }
  request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
  request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

  try {
    doDispatch(request, response);
  }
  finally {
    if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
      // Restore the original attribute snapshot, in case of an include.
      if (attributesSnapshot != null) {
        restoreAttributesAfterInclude(request, attributesSnapshot);
      }
    }
  }
}
```

**doDispatch()方法源码：该过程是负责请求的真正处理**

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  HttpServletRequest processedRequest = request;
  HandlerExecutionChain mappedHandler = null;
  boolean multipartRequestParsed = false;

  WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

  try {
    ModelAndView mv = null;
    Exception dispatchException = null;

    try {
      // 1. 判断用户的请求是否为文件上传，通过MultipartResolver解析
      processedRequest = checkMultipart(request);
      multipartRequestParsed = (processedRequest != request);

      // 2. 通过HandlerMapping映射处理器和拦截器包装成一个HandlerExecutionChain
      mappedHandler = getHandler(processedRequest);
      if (mappedHandler == null || mappedHandler.getHandler() == null) {
        noHandlerFound(processedRequest, response);
        return;
      }

      // 3. 将处理器包装成相应的适配器
      HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

      // 省略此处代码

      // 4. 执行处理器相关的拦截器的预处理
      if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
      }

      // 5. 由适配器执行处理器
      mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

      if (asyncManager.isConcurrentHandlingStarted()) {
        return;
      }

      applyDefaultViewName(request, mv);

      // 6. 拦截器后处理
      mappedHandler.applyPostHandle(processedRequest, response, mv);
    }
    catch (Exception ex) {
      dispatchException = ex;
    }

    // 7. 视图渲染
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
  }
  // 此处省略部分代码
}
```




## 3. doDispatch()流程解析



![SpringMVC工作流程](https://raw.githubusercontent.com/zchdjb/zchdjb.github.io/master/images/springmvc.png)



核心架构的具体流程步骤如下：

1、  首先用户发送请求——>DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他的解析器进行处理，作为统一访问点，进行全局的流程控制；

2、  DispatcherServlet——>HandlerMapping， HandlerMapping将会把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器）对象，通过这种策略模式，很容易添加新的映射策略；

3、  DispatcherServlet——>HandlerAdapter，HandlerAdapter将会把处理器包装为适配器，从而支持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；

4、  HandlerAdapter——>处理器功能处理方法的调用，HandlerAdapter将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理；并返回一个ModelAndView对象（包含模型数据、逻辑视图名）；

5、  ModelAndView的逻辑视图名——> ViewResolver， ViewResolver将把逻辑视图名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；

6、  View——>渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术；

7、返回控制权给DispatcherServlet，由DispatcherServlet返回响应给用户，到此一个流程结束。