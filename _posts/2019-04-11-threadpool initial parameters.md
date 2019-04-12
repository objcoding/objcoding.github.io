---
layout: post
title: "你都理解创建线程池的参数吗？"
categories: Java
tags: 多线程
author: zch
---

* content
{:toc}
多线程可以说是面试官最喜欢拿来问的题目之一了，可谓是老生之常谈，不管你是新手还是老司机，我相信你一定会在面试过程中遇到过有关多线程的一些问题。那我现在就充当一次面试官，我来问你：

**现有一个线程池，参数corePoolSize = 5，maximumPoolSize = 10，BlockingQueue阻塞队列长度为5，此时有4个任务同时进来，问：线程池会创建几条线程？**

**如果4个任务还没处理完，这时又同时进来2个任务，问：线程池又会创建几条线程还是不会创建？**

**如果前面6个任务还是没有处理完，这时又同时进来5个任务，问：线程池又会创建几条线程还是不会创建？**

如果你此时一脸懵逼，请不要慌，问题不大。

![dont_panic](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/dont_panic.jpeg)








## 创建线程池的构造方法的参数都有哪些？

要回答这个问题，我们需要从创建线程池的参数去找答案：

java.util.concurrent.ThreadPoolExecutor#ThreadPoolExecutor：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 || maximumPoolSize <= 0 || maximumPoolSize < corePoolSize || keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ? null : AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

创建线程池一共有7个参数，从源码可知，corePoolSize和maximumPoolSize都不能小于0，且核心线程数不能大于最大线程数。

下面我来解释一下这7个参数的用途：

### corePoolSize

线程池核心线程数量，核心线程不会被回收，即使没有任务执行，也会保持空闲状态。如果线程池中的线程少于此数目，则在执行任务时创建。

### maximumPoolSize

池允许最大的线程数，当线程数量达到corePoolSize，且workQueue队列塞满任务了之后，继续创建线程。

### keepAliveTime

超过corePoolSize之后的“临时线程”的存活时间。

### unit

keepAliveTime的单位。

### workQueue

当前线程数超过corePoolSize时，新的任务会处在等待状态，并存在workQueue中，BlockingQueue是一个先进先出的阻塞式队列实现，底层实现会涉及Java并发的AQS机制，有关于AQS的相关知识，我会单独写一篇，敬请期待。

### threadFactory

创建线程的工厂类，通常我们会自顶一个threadFactory设置线程的名称，这样我们就可以知道线程是由哪个工厂类创建的，可以快速定位。

### handler

线程池执行拒绝策略，当线数量达到maximumPoolSize大小，并且workQueue也已经塞满了任务的情况下，线程池会调用handler拒绝策略来处理请求。

系统默认的拒绝策略有以下几种：
1. AbortPolicy：为线程池默认的拒绝策略，该策略直接抛异常处理。
2. DiscardPolicy：直接抛弃不处理。
3. DiscardOldestPolicy：丢弃队列中最老的任务。
4. CallerRunsPolicy：将任务分配给当前执行execute方法线程来处理。

我们还可以自定义拒绝策略，只需要实现RejectedExecutionHandler接口即可，友好的拒绝策略实现有如下：
1. 将数据保存到数据，待系统空闲时再进行处理
2. 将数据用日志进行记录，后由人工处理



现在我们回到刚开始的问题就很好回答了：

**线程池corePoolSize=5，线程初始化时不会自动创建线程，所以当有4个任务同时进来时，执行execute方法会新建【4】条线程来执行任务；**

**前面的4个任务都没完成，现在又进来2个队列，会新建【1】条线程来执行任务，这时poolSize=corePoolSize，还剩下1个任务，线程池会将剩下这个任务塞进阻塞队列中，等待空闲线程执行；**

**如果前面6个任务还是没有处理完，这时又同时进来了5个任务，此时还没有空闲线程来执行新来的任务，所以线程池继续将这5个任务塞进阻塞队列，但发现阻塞队列已经满了，核心线程也用完了，还剩下1个任务不知道如何是好，于是线程池只能创建【1】条“临时”线程来执行这个任务了；**

**这里创建的线程用“临时”来描述还是因为它们不会长期存在于线程池，它们的存活时间为keepAliveTime，此后线程池会维持最少corePoolSize数量的线程。**

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/threadpool.png)

## 为什么不建议使用Executors创建线程池？

JDK为我们提供了Executors线程池工具类，里面有默认的线程池创建策略，大概有以下几种：

1. FixedThreadPool：线程池线程数量固定，即corePoolSize和maximumPoolSize数量一样。
2. SingleThreadPool：单个线程的线程池。
3. CachedThreadPool：初始核心线程数量为0，最大线程数量为Integer.MAX_VALUE，线程空闲时存活时间为60秒，并且它的阻塞队列为SynchronousQueue，它的初始长度为0，这会导致任务每次进来都会创建线程来执行，在线程空闲时，存活时间到了又会释放线程资源。
4. ScheduledThreadPool：创建一个定长的线程池，而且支持定时的以及周期性的任务执行，类似于Timer。

用Executors工具类虽然很方便，我依然不推荐大家使用以上默认的线程池创建策略，阿里巴巴开发手册也是强制不允许使用Executors来创建线程池，我们从JDK源码中寻找一波答案：

java.util.concurrent.Executors：
```java
// FixedThreadPool
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

// SingleThreadPool
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

// CachedThreadPool
public static ExecutorService newCachedThreadPool() {
    // 允许创建线程数为Integer.MAX_VALUE
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

// ScheduledThreadPool
public ScheduledThreadPoolExecutor(int corePoolSize) {
    // 允许创建线程数为Integer.MAX_VALUE
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```
```java
public LinkedBlockingQueue() {
    // 允许队列长度最大为Integer.MAX_VALUE
    this(Integer.MAX_VALUE);
}
```

从JDK源码可看出，Executors工具类无非是把一些特定参数进行了封装，并提供一些方法供我们调用而已，我们并不能灵活地填写参数，**策略过于简单，不够友好**。

CachedThreadPool和ScheduledThreadPool最大线程数为Integer.MAX_VALUE，如果线程无限地创建，会造成OOM异常。

LinkedBlockingQueue基于链表的FIFO队列，是无界的，默认大小是Integer.MAX_VALUE，因此FixedThreadPool和SingleThreadPool的阻塞队列长度为Integer.MAX_VALUE，如果此时队列被无限地堆积任务，会造成OOM异常。



