---
layout: post
title: "关于线程池你不得不知道的一些设置"
categories: Java
tags: 线程池
author: 张乘辉
---

* content
{:toc}
看完我上一篇文章「你都理解创建线程池的参数吗？」之后，当遇到这种问题，你觉得你完全能够唬住面试官了，50k轻松到手。殊不知，要是面试官此刻给你来个反杀：

**初始化线程池时可以预先创建线程吗？线程池的核心线程可以被回收吗？为什么？**

如果此刻你一脸懵逼，这个要慌，问题很大，50k马上变5k。

![](https://gitee.com/objcoding/md-picture/raw/master/img/threadpool_7.jpg)











有细心的网友早就想到了这个问题：

![](https://gitee.com/objcoding/md-picture/raw/master/img/threadpool_2.jpg)

![](https://gitee.com/objcoding/md-picture/raw/master/img/threadpool_6.jpg)



在ThreadPoolExecutor线程池中，还有一些不常用的设置。**我建议如果您在应用场景中没有特殊的要求，就不需要使用这些设置。**



## 初始化线程池时可以预先创建线程吗？

### prestartAllCoreThreads

初始化线程池时是可以预先创建线程的，初始化线程池后，再调用prestartAllCoreThreads()方法，即可预先创建corePoolSize数量的核心线程，我们看源码：

```java
public int prestartAllCoreThreads() {
    int n = 0;
    while (addWorker(null, true))
        ++n;
    return n;
}
```

```java
private boolean addWorker(Runnable firstTask, boolean core) {
  // ..
}
```

addWorker方法目的是在线程池中添加任务并执行，**如果task为空，线程获取任务执行时调用getTask()方法，该方法从blockingQueue阻塞队列中阻塞获取任务执行，因此线程不会释放，留存在线程池中**，如果core=true，说明任务只能利用核心线程来执行。

所以该方法会在线程池总预先创建没有任务执行的线程，数量为corePoolSize。

下面我们测试一下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/threadpool_4.png)

从测试结果来看，线程池中已经预先创建了corePoolSize数量的空闲线程。



### prestartCoreThread

prestartCoreThread()同样可以预先创建线程，只不过该方法只会与创建1条线程，我们来看源码：

```java
public boolean prestartCoreThread() {
    return workerCountOf(ctl.get()) < corePoolSize &&
        addWorker(null, true);
}
```

从方法源码可知，如果此时工作线程数量小于corePoolSize，那么就调用addWorker创建1条空闲核心线程。

下面我们测试一下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/threadpool_5.png)

从测试结果来看，线程池中已经预先创建了1条空闲线程。



## 线程池的核心线程可以被回收吗？

你可能会想到将corePoolSize的数量设置为0，从而线程池的所有线程都是“临时”的，只有keepAliveTime存活时间，你的思路也许时正确的，但你有没有想过一个很严重的后果，corePoolSize=0时，任务需要填满阻塞队列才会创建线程来执行任务，阻塞队列有设置长度还好，如果队列长度无限大呢，你就等着OOM异常吧，所以用这种设置行为并不是我们所需要的。

有没有什么设置可以回收核心线程呢？

### allowCoreThreadTimeOut


ThreadPoolExecutor有一个私有成员变量：


```java
private volatile boolean allowCoreThreadTimeOut;
```

如果allowCoreThreadTimeOut=true，核心线程在规定时间内会被回收。

上面我也说了，当线程空闲时会从blockingQueue阻塞队列中阻塞获取任务执行，所以我们来看看是保证核心线程不被销毁的，我们直接定位到源码部位：

java.util.concurrent.ThreadPoolExecutor#getTask：

```java
boolean timedOut = false; // Did the last poll() time out?
for (;;) {
    // Are workers subject to culling?
    boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

    try {
        Runnable r = timed ?
            workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
        workQueue.take();
        if (r != null)
            return r;
        timedOut = true;
    } catch (InterruptedException retry) {
        timedOut = false;
    }
}
```

这里的关键值timed，如果allowCoreThreadTimeOut=true或者此时工作线程大于corePoolSize，timed=true，如果timed=true，会调用poll()方法从阻塞队列中获取任务，否则调用take()方法获取任务。

下面我来解释这两个方法：

1. poll(long timeout, TimeUnit unit)：从BlockingQueue取出一个任务，如果不能立即取出，则可以等待timeout参数的时间，如果超过这个时间还不能取出任务，则返回null；
2. take()：从blocking阻塞队列取出一个任务，如果BlockingQueue为空，阻断进入等待状态直到BlockingQueue有新的任务被加入为止。

到这里，我们就很好地解释了，**当allowCoreThreadTimeOut=true或者此时工作线程大于corePoolSize时，线程调用BlockingQueue的poll方法获取任务，若超过keepAliveTime时间，则返回null，timedOut=true，则getTask会返回null，线程中的runWorker方法会退出while循环，线程接下来会被回收。**

下面我们测试一下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/threadpool_3.png)

可以看到，核心线程被回收了。

