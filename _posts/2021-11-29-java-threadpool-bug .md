---
layout: post
title: "一个 JDK 线程池 BUG 引发的 GC 机制思考"
categories: Java
tags: 线程池 GC
author: 空无
---

* content
{:toc}


前几天在帮同事排查生产一个线上偶发的线程池错误

逻辑很简单，线程池执行了一个带结果的异步任务。但是最近有偶发的报错：

```
java.util.concurrent.RejectedExecutionException: Task java.util.concurrent.FutureTask@a5acd19 rejected from java.util.concurrent.ThreadPoolExecutor@30890a38[Terminated, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 0]
```

本文中的模拟代码已经问题都是在HotSpot java8 (1.8.0_221)版本下模拟&出现的。











下面是模拟代码，通过Executors.newSingleThreadExecutor创建一个单线程的线程池，然后在调用方获取Future的结果：

```java
public class ThreadPoolTest {

    public static void main(String[] args) {
        final ThreadPoolTest threadPoolTest = new ThreadPoolTest();
        for (int i = 0; i < 8; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    while (true) {

                        Future<String> future = threadPoolTest.submit();
                        try {
                            String s = future.get();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } catch (ExecutionException e) {
                            e.printStackTrace();
                        } catch (Error e) {
                            e.printStackTrace();
                        }
                    }
                }
            }).start();
        }

        //子线程不停gc，模拟偶发的gc
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    System.gc();
                }
            }
        }).start();
    }

    /**
     * 异步执行任务
     * @return
     */
    public Future<String> submit() {
        //关键点，通过Executors.newSingleThreadExecutor创建一个单线程的线程池
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        FutureTask<String> futureTask = new FutureTask(new Callable() {
            @Override
            public Object call() throws Exception {
                Thread.sleep(50);
                return System.currentTimeMillis() + "";
            }
        });
        executorService.execute(futureTask);
        return futureTask;
    }

}
```



## 分析&疑问



第一个思考的问题是：线程池为什么关闭了，代码中并没有手动关闭的地方。看一下`Executors.newSingleThreadExecotor`的源码实现：

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

这里创建的实际上是一个`FinalizableDelegatedExecutorService`，这个包装类重写了`finalize`函数，也就是说这个类会在被GC回收之前，先执行线程池的shutdown方法。

问题来了，**GC只会回收不可达（unreachable）的对象**，在`submit`函数的栈帧未执行完出栈之前，`executorService`应该是可达的才对。

对于此问题，先抛出结论：

**当对象仍存在于作用域（stack frame）时，`finalize`也可能会被执行**

oracle jdk文档中有一段关于finalize的介绍：

> A reachable object is any object that can be accessed in any potential continuing computation from any live thread.
>
> Optimizing transformations of a program can be designed that reduce the number of objects that are reachable to be less than those which would naively be considered reachable. For example, a Java compiler or code generator may choose to set a variable or parameter that will no longer be used to null to cause the storage for such an object to be potentially reclaimable sooner.

**大概意思是：可达对象(reachable object)是可以从任何活动线程的任何潜在的持续访问中的任何对象；java编译器或代码生成器可能会对不再访问的对象提前置为null，使得对象可以被提前回收。**

也就是说，在jvm的优化下，可能会出现对象不可达之后被提前置空并回收的情况。

举个例子来验证一下（摘自[https://stackoverflow.com/questions/24376768/can-java-finalize-an-object-when-it-is-still-in-scope）](https://link.segmentfault.com/?enc=mRCPxtURWwGq0b3ttFbsfA%3D%3D.wrRPzOsRCMDaAZjRfHTTJ17FW1LdVvEqn%2FEteXg5PJoGkQy78mzD2jOl4EskyJzWKs5Z4kpClBbq4uOPjQenFMfCcdFhl%2Bl%2BtSKa4%2B3LSiQH23E7GdzRAMUxCep9NdmsNDsEX8SrZ6wMWZX4MdVL7Q%3D%3D)：



```java
class A {
    @Override protected void finalize() {
        System.out.println(this + " was finalized!");
    }

    public static void main(String[] args) throws InterruptedException {
        A a = new A();
        System.out.println("Created " + a);
        for (int i = 0; i < 1_000_000_000; i++) {
            if (i % 1_000_00 == 0)
                System.gc();
        }
        System.out.println("done.");
    }
}

//打印结果
Created A@1be6f5c3
A@1be6f5c3 was finalized!//finalize方法输出
done.
```



从例子中可以看到，如果a在循环完成后已经不再使用了，则会出现先执行finalize的情况；虽然从对象作用域来说，方法没有执行完，栈帧并没有出栈，但是还是会被提前执行。

现在来增加一行代码，在最后一行打印对象a，让编译器/代码生成器认为后面有对象a的引用。

```
...
System.out.println(a);

//打印结果
Created A@1be6f5c3
done.
A@1be6f5c3
```

从结果上看，finalize方法都没有执行（因为main方法执行完成后进程直接结束了），更不会出现提前finalize的问题了

基于上面的测试结果，再测试一种情况，在循环之前先将对象a置为null，并且在最后打印保持对象a的引用

```java
A a = new A();
System.out.println("Created " + a);
a = null;//手动置null
for (int i = 0; i < 1_000_000_000; i++) {
    if (i % 1_000_00 == 0)
        System.gc();
}
System.out.println("done.");
System.out.println(a);

//打印结果
Created A@1be6f5c3
A@1be6f5c3 was finalized!
done.
null
```



从结果上看，手动置null的话也会导致对象被提前回收，虽然在最后还有引用，但此时引用的也是null了。



现在再回到上面的线程池问题，根据上面介绍的机制，在分析没有引用之后，对象会被提前finalize

可在上述代码中，return之前明明是有引用的`executorService.execute(futureTask)`，为什么也会提前finalize呢？

猜测可能是由于在execute方法中，会调用threadPoolExecutor，会创建并启动一个新线程，这时会发生一次主动的线程切换，导致在活动线程中对象不可达

结合上面Oracle Jdk文档中的描述“可达对象(reachable object)是可以从任何活动线程的任何潜在的持续访问中的任何对象”，可以认为可能是因为一次显示的线程切换，对象被认为不可达了，导致线程池被提前finalize了

下面来验证一下猜想：



```java
//入口函数
public class FinalizedTest {
    public static void main(String[] args) {
        final FinalizedTest finalizedTest = new FinalizedTest();
        for (int i = 0; i < 8; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    while (true) {
                        TFutureTask future = finalizedTest.submit();
                    }
                }
            }).start();
        }
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    System.gc();
                }
            }
        }).start();
    }
    public TFutureTask submit(){
        TExecutorService TExecutorService = Executors.create();
        TExecutorService.execute();
        return null;
    }
}

//Executors.java，模拟juc的Executors
public class Executors {
    /**
     * 模拟Executors.createSingleExecutor
     * @return
     */
    public static TExecutorService create(){
        return new FinalizableDelegatedTExecutorService(new TThreadPoolExecutor());
    }

    static class FinalizableDelegatedTExecutorService extends DelegatedTExecutorService {

        FinalizableDelegatedTExecutorService(TExecutorService executor) {
            super(executor);
        }
        
        /**
         * 析构函数中执行shutdown，修改线程池状态
         * @throws Throwable
         */
        @Override
        protected void finalize() throws Throwable {
            super.shutdown();
        }
    }

    static class DelegatedTExecutorService extends TExecutorService {

        protected TExecutorService e;

        public DelegatedTExecutorService(TExecutorService executor) {
            this.e = executor;
        }

        @Override
        public void execute() {
            e.execute();
        }

        @Override
        public void shutdown() {
            e.shutdown();
        }
    }
}

//TThreadPoolExecutor.java，模拟juc的ThreadPoolExecutor
public class TThreadPoolExecutor extends TExecutorService {

    /**
     * 线程池状态，false：未关闭，true已关闭
     */
    private AtomicBoolean ctl = new AtomicBoolean();

    @Override
    public void execute() {
        //启动一个新线程，模拟ThreadPoolExecutor.execute
        new Thread(new Runnable() {
            @Override
            public void run() {

            }
        }).start();
        //模拟ThreadPoolExecutor，启动新建线程后，循环检查线程池状态，验证是否会在finalize中shutdown
        //如果线程池被提前shutdown，则抛出异常
        for (int i = 0; i < 1_000_000; i++) {
            if(ctl.get()){
                throw new RuntimeException("reject!!!["+ctl.get()+"]");
            }
        }
    }

    @Override
    public void shutdown() {
        ctl.compareAndSet(false,true);
    }
}
```



执行若干时间后报错：

```
Exception in thread "Thread-1" java.lang.RuntimeException: reject!!![true]
```

从错误上来看，“线程池”同样被提前shutdown了，那么一定是由于新建线程导致的吗？

下面将新建线程修改为`Thread.sleep`测试一下：

```java
//TThreadPoolExecutor.java，修改后的execute方法
public void execute() {
    try {
        //显式的sleep 1 ns，主动切换线程
        TimeUnit.NANOSECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    //模拟ThreadPoolExecutor，启动新建线程后，循环检查线程池状态，验证是否会在finalize中shutdown
    //如果线程池被提前shutdown，则抛出异常
    for (int i = 0; i < 1_000_000; i++) {
        if(ctl.get()){
            throw new RuntimeException("reject!!!["+ctl.get()+"]");
        }
    }
}
```



执行结果一样是报错:

```
Exception in thread "Thread-3" java.lang.RuntimeException: reject!!![true]
```

**由此可得，如果在执行的过程中，发生一次显式的线程切换，则会让编译器/代码生成器认为外层包装对象不可达**

## 总结



虽然GC只会回收不可达GC ROOT的对象，但是在编译器（没有明确指出，也可能是JIT）/代码生成器的优化下，可能会出现对象提前置null，或者线程切换导致的“提前对象不可达”的情况。

所以如果想在finalize方法里做些事情的话，一定在最后显示的引用一下对象（toString/hashcode都可以），保持对象的可达性（reachable）

上面关于线程切换导致的对象不可达，没有官方文献的支持，只是个人一个测试结果，如有问题欢迎指出

**综上所述，这种回收机制并不是JDK的bug，而算是一个优化策略，提前回收而已；但`Executors.newSingleThreadExecutor`的实现里通过finalize来自动关闭线程池的做法是有Bug的，在经过优化后可能会导致线程池的提前shutdown，从而导致异常。**

线程池的这个问题，在JDK的论坛里也是一个公开但未解决状态的问题[https://bugs.openjdk.java.net/browse/JDK-8145304](https://link.segmentfault.com/?enc=jimTsDI3dBl%2BqApeEcIZoA%3D%3D.tEAYbWLKbbecEtKcfP6PguhikfRYRHQovt0Yy%2BWwefQ2ZRsVMF806j%2Beti6qEWKTBOSrE6%2BbnU4YaSwm7TVmSw%3D%3D)。

不过在JDK11下，该问题已经被修复：



```java
JUC  Executors.FinalizableDelegatedExecutorService
public void execute(Runnable command) {
    try {
        e.execute(command);
    } finally { reachabilityFence(this); }
}
```



