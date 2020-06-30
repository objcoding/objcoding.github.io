---
layout: post
title: "Spring 异步实现原理与实战分享"
categories: Spring
tags: async
author: zch
---

* content
{:toc}
最近因为全链路压测项目需要对用户自定义线程池 Bean 进行适配工作，我们知道全链路压测的核心思想是对流量压测进行标记，因此我们需要给压测的流量请求进行打标，并在链路中进行传递，那么问题来了，如果项目中使用了多线程处理业务，就会造成父子线程间无法传递压测打标数据，不过可以利用阿里开源的 ttl 解决这个问题。

全链路压测项目的宗旨就是不让用户感知这个项目的存在，因此我们不可能让用户去对其线程池进行改造的，我们需要主动去适配用户自定义的线程池。

在适配过程的过程中无非就是将线程池替换成 ttl 去解决，可通过代理或者替换 Bean 的方式实现，这方面不是本文的内容，本文主要是深入 Spring 异步实现的原理，让大家对 Spring 异步编程不再陌生！





## 运行原理分析

过一遍源码分析，才能知道其中的一些细节原理，这也是不可避免的过程，虽然我也不想在文章中贴过多的源码，但如果不从源码中得出原因，很可能你会知其然不知其所以然。下面就尽量跟着源码走一遍它的运行机制是怎么样的，我把我自己的理解也会尽量详细地描述出来，在这里我会将其关联的源码贴出来分析，这些源码都有其相互关联性，可能你看到后面还会回来再看一遍。



### 注册通知器过程

开启 Spring 异步编程之需要一个注解即可：

```java
@EnableAsync
```

Springboot 中有非常多 @Enable* 的注解，其目的是显式开启某一个功能特性，这也是一个非常典型的编程模型。

@EnableAsync 注解注入了一个 AsyncConfigurationSelector 类，这个类目的就是为了注入 ProxyAsyncConfiguration 自动配置类，它的父类 AbstractAsyncConfiguration 做了件事情：

org.springframework.scheduling.annotation.AbstractAsyncConfiguration#setConfigurers

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611195610.png)

我们可以实现 AsyncConfigurer 接口的方式去自定义一个线程池 Bean，这个后面会会讲到，源码所示，这里目的是为了这个 bean，并将其定义的线程池对象和异常处理对象保存到 AsyncConfiguration 中，用于创建 AsyncAnnotationBeanPostProcessor 。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611195946.png)

这两个对象后面源码分析会再次遇上。

而这个配置类就是为了注册一个名为 AsyncAnnotationBeanPostProcessor 的 bean，如其名，它是一个 BeanPostProcessor 处理器，它的类继承结构如下所示：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611195400.png)

从类继承结构可以看出，AsyncAnnotationBeanPostProcessor 实现了 BeanPostProcessor 和 BeanFactoryAware，因此 AsyncAnnotationBeanPostProcessor 会在 setBeanFactory 方法中做了 Spring 异步编程中最为重要的一步，创建一个针对 @Async 注解的通知器 AsyncAnnotationAdvisor（叫做切面貌似也可以），这个通知器主要用于拦截被 @Async 注解的方法。同时，bean 实例初始化过程会被  AsyncAnnotationBeanPostProcessor 拦截处理，处理过程会将符合条件的 bean 注册 AsyncAnnotationAdvisor ：

org.springframework.aop.framework.AbstractAdvisingBeanPostProcessor#postProcessAfterInitialization

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611220044.png)



### 创建通知器过程

接下来我们就分析 AsyncAnnotationAdvisor 是如何创建的。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611205738.png)

AsyncAnnotationAdvisor 实现了 PointcutAdvisor 接口，因此需要同时实现 getPointcut 和 getAdvice 方法，而这两个方法的实际内容有以上红框创建实现。

**到这里我们已经知道，Spring 的异步实现原理，是利用 Spring AOP 切面编程实现的，通过 BeanPostProcessor 拦截处理符合条件的 bean，并将切面织入，实现切面增强处理。**

Spring AOP 的编程核心概念：

> 1. Advice：通知，切面的一种实现，可以完成简单的织入功能。通知定义了增强代码切入到目标代码的时间点，是目标方法执行之前执行，还是执行之后执行等。切入点定义切入的位置，通知定义切入的时间；
> 2. Pointcut：切点，切入点指切面具体织入的方法；
> 3. Advisor：切面的另一种实现，能够将通知以更为复杂的方式织入到目标对象中，是将通知包装为更复杂切面的装配器。

因此我们需要创建一个切面和切入点：

- buildAdvice：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611210607.png)

buildAdvice 方法可知，切面是一个 AnnotationAsyncExecutionInterceptor 类，该类实现了 MethodInterceptor 接口，其 invoke 方法即为拦截处理的核心源码，后面会进行详细分析。

- buildPointcut:

从 AsyncAnnotationAdvisor 构造器中可以看出，buildPointcut 方法目的就是为了创建 @Async 注解的切入点。



### 通知器拦截处理过程

前面我们已经知道，拦截切面是一个 AnnotationAsyncExecutionInterceptor 类，我们直接定位到 invoke 方法一探究竟：

org.springframework.aop.interceptor.AsyncExecutionInterceptor#invoke

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611224149.png)

拦截处理的核心逻辑就是这么简单，也没啥好分析的，无非就是匹配方法指定的线程池，接着构建执行单元 Callable，最后调用 doSubmit 方法执行。

### 如何匹配线程池？

重点在于如何匹配线程池，这也是后面实战分析的重点内容，因此我们需要在这里详细分析匹配线程池的一些策略细节。

org.springframework.aop.interceptor.AsyncExecutionAspectSupport#determineAsyncExecutor

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611225209.png)

getExecutorQualifier 方法目的是获取 @Async 注解上的 value 值，value 值即线程池 Bean 的名称，如果获取到的 targetExecutor 不是 Spring 类型的线程池，则使用 TaskExecutorAdapter 进行适配，这也是为什么我们直接创建 Executor 类型的线程池 Spring 也是支持的原因。

从以上源码逻辑可看出如果我们使用 @Async 注解时 value 值为空，Spring 就会使用 defaultExecutor ，defaultExecutor 是什么时候赋值的呢？上面内容已经有提及，在 buildAdvice 方法创建 AnnotationAsyncExecutionInterceptor 时 调用了其 configure 方法，如下：

org.springframework.aop.interceptor.AsyncExecutionAspectSupport#configure

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200614202430.png)

原来当 defaultExecutor 和 exceptionHandler 是当初从 ProxyAsyncConfiguration 中获取用户自定义的 AsyncConfigurer 实现类而来的，那么如果 defaultExecutor 不存在怎么办？从源码可看出，defaultExecutor 其实是一个 SingletonSupplier 类型，如果调用 get 方法不存在，则使用默认值，默认值为：

```java
() -> getDefaultExecutor(this.beanFactory);
```

org.springframework.aop.interceptor.AsyncExecutionAspectSupport#getDefaultExecutor

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200614203436.png)

注意第一个红框的注释，此时 Spring 寻找默认的线程池 Bean 为指定 Spring 的 TaskExecutor 类型，并非 Executor 类型，如果 Bean 容器中没有找到  TaskExecutor 类型的 Bean，则继续寻找默认为以下名称的 Bean：

```java
public static final String DEFAULT_TASK_EXECUTOR_BEAN_NAME = "taskExecutor";
```

那么如果都没有找到怎么办呢？在这个方法直接返回 null 了，AsyncExecutionInterceptor 类覆写了 这个方法：

org.springframework.aop.interceptor.AsyncExecutionInterceptor#getDefaultExecutor
![](https://gitee.com/objcoding/md-picture/raw/master/img/20200614203955.png)

如果没有找到，则直接创建一个 SimpleAsyncTaskExecutor 类作为 @Async 注解底层使用的线程池。

**从匹配线程池源码得知，如果你创建的线程池 Bean 非TaskExecutor 类型并且没有使用实现 AsyncConfigurer 接口方式创建线程池，就需要主动指定线程池 Bean 名称，否则 Spring 会使用默认策略。**



### 总结

利用 BeanPostProcessor 机制在 Bean 初始化过程中创建一个 AsyncAnnotationAdvisor 切面，并且符合条件的 Bean 生成代理对象并将 AsyncAnnotationAdvisor 切面添加到代理中。

可以看出 Spring 的很多功能都是围绕着 Spring IOC 和 AOP 实现的。



## Spring 默认线程池策略分析

有时候为了方便，我们不自定义创建线程池 bean 时，Spring 默认会为我们提供什么样的线程池呢？

我们先来看下结果：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200528165656.png)

很奇怪，明明我们都没有在项目中自定义线程池 Bean，按照以上源码的分析结果来看，此时 Spring 选择的是 SimpleAsyncTaskExecutor 才对，莫非是 super#getDefaultExecutor 方法找到了线程池 Bean？

从以上截图确实是找到了，而且类型还是 ThreadPoolTaskExecutor 类型的，那可以推断出 Spring 一定是在某个地方创建了一个 ThreadPoolTaskExecutor 类型的 Bean。

果然，在 spring-boot-autoconfigure 2.1.3.RELEASE 中，会在 org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration 中自动创建一个默认的 ThreadPoolTaskExecutor bean，getDefaultExecutor 方法会在容器中找到这个bean，并将其作为默认的 @Async 注解的执行线程池。

这里我为什么要标注版本呢？因为某些低版本的 spring-boot-autoconfigure，是没有 TaskExecutionAutoConfiguration 的，此时 Spring 就会选择 SimpleAsyncTaskExecutor。

org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200611163322.png)

从以上源码可以看出，默认的线程池的参数还可以手动在 properties 中配置，这意味着不需要主动创建线程池的情况下，也可以通过 properties 配置文件更改线程池相关参数。



## 创建线程池 Bean 的几种方式

1、直接创建一个 Bean 的方式，这貌似是最多人使用的方式，可以创建多个线程池 Bean，使用时指定线程池 Bean 名称：

```java
@Bean("myTaskExecutor_1")
public Executor getThreadPoolTaskExecutor1() {
  final ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
  // set ...
  return executor;
}

@Bean("myTaskExecutor_2")
public Executor getThreadPoolTaskExecutor2() {
  final ThreadPoolExecutor executor = new ThreadPoolExecutor();
  // set ...
  return executor;
}
```

2、实现 AsyncConfigurer 接口方式：

```java
@Component
public class AsyncConfigurerTest implements AsyncConfigurer {

  private static final Logger LOGGER = LoggerFactory.getLogger(AsyncConfigurerTest.class);

  @Override
  public Executor getAsyncExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    // set ...
    return executor;
  }

  @Override
  public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return (ex, method, params) -> {
      LOGGER.info("Exception message:{}", ex.getMessage(), ex);
      LOGGER.info("Method name:{}", method.getName());
      for (Object param : params) {
        LOGGER.info("Parameter value:{}", param);
      }
    };
  }
}
```

这种方式可以方便定义异常处理的逻辑，不过从源码分析可以看出，项目中只能存在一个 AsyncConfigurer 的配置，意味着我们只能通过 AsyncConfigurer 配置一个自定义的线程池 Bean。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200614224541.png)

3、利用 spring-boot-autoconfigure 在 properties 配置线程池参数：

前面讲到了 Spring 默认线程池策略，这里利用 spring-boot-autoconfigure 默认创建一个 ThreadPoolTaskExecutor，通过  properties 自定义线程池相关参数。

这个方式的缺点就是类型固定为 ThreadPoolTaskExecutor，且只能有一个线程池。

*注：以上所有原理分析与实战结果都是基于 Spring 5.1.5.RELEASE 版本。*