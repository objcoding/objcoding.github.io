---
layout: post
title: "SpringMVC处理请求过程"
categories: SpringMVC
tags: Spring MVC
author: zch
---

* content
{:toc}


上文也说过，HttpServletBean继承了Httpservlet，因此它也遵循Servlet流程来处理用户请求。





## FrameworkServlet请求处理

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

## DispatcherServlet请求处理

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




## doDispatch()流程解析

参考原文地址：http://jinnianshilongnian.iteye.com/blog/1594806

![SpringMVC工作流程](https://gitee.com/objcoding/md-picture/raw/master/img/springmvc.png)



核心架构的具体流程步骤如下：

1、  首先用户发送请求——>DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他的解析器进行处理，作为统一访问点，进行全局的流程控制；

2、  DispatcherServlet——>HandlerMapping， HandlerMapping将会把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器）对象，通过这种策略模式，很容易添加新的映射策略；

3、  DispatcherServlet——>HandlerAdapter，HandlerAdapter将会把处理器包装为适配器，从而支持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；

4、  HandlerAdapter——>处理器功能处理方法的调用，HandlerAdapter将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理；并返回一个ModelAndView对象（包含模型数据、逻辑视图名）；

5、  ModelAndView的逻辑视图名——> ViewResolver， ViewResolver将把逻辑视图名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；

6、  View——>渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术；

7、返回控制权给DispatcherServlet，由DispatcherServlet返回响应给用户，到此一个流程结束。