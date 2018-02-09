---
layout: post
title: "浅谈多线程中的synchronized和volatile关键字"
categories: Java
tags: 多线程 synchronized volatile
author: zch
---

* content
{:toc}
synchronized主要为线程提供同步功能，而volatile主要是设置参数的可见性。





## synchronized

在并发编程过程中，很多会涉及到线程安全问题，“非线程安全”指的是多个线程访问同一个对象中的实例变量进行并发访问时发生的“脏读”，而“线程安全”指的是多个线程同时访问的变量为私有变量，并不会共享，不会产生脏读现象，也就是说如果变量在方法内，就不会产生“非线程安全”的问题了。为了解决示例变量可能出现的“非线程安全”的问题，我们需多线程关键字synchronized。

java中，每一个对象有且仅有一个同步锁。这也意味着，同步锁是依赖于对象而存在。当我们调用某对象的synchronized方法时，就获取了该对象的同步锁。例如，synchronized(obj)就获取了“obj这个对象”的同步锁。不同线程对同步锁的访问

是互斥的。也就是说，某时间点，对象的同步锁只能被一个线程获取到！通过同步锁，我们就能在多线程中，实现对“对象/方法”的互斥访问。 例如，现在有两个线程A和线程B，它们都会访问“对象obj的同步锁”。假设，在某一时刻，线程A获取到“obj的同步锁”并在执行一些操作；而此时，线程B也企图获取“obj的同步锁” —— 线程B会获取失败，它必须等待，直到线程A释放了“该对象的同步锁”之后线程B才能获取到“obj的同步锁”从而才可以运行。

### synchronized语句块

参考原文地址：http://blog.csdn.net/luoweifu/article/details/46613015

我们来看一个例子：

```java
/**
 * 同步线程
 */
class SyncThread implements Runnable {
   private static int count;

   public SyncThread() {
      count = 0;
   }

   public  void run() {
      synchronized(this) {
         for (int i = 0; i < 5; i++) {
            try {
               System.out.println(Thread.currentThread().getName() + ":" + (count++));
               Thread.sleep(100);
            } catch (InterruptedException e) {
               e.printStackTrace();
            }
         }
      }
   }

   public int getCount() {
      return count;
   }
}
```

调用：

```java
SyncThread syncThread = new SyncThread();
Thread thread1 = new Thread(syncThread, "SyncThread1");
Thread thread2 = new Thread(syncThread, "SyncThread2");
thread1.start();
thread2.start();
```

执行结果：

```java
SyncThread1:0 
SyncThread1:1 
SyncThread1:2 
SyncThread1:3 
SyncThread1:4 
SyncThread2:5 
SyncThread2:6 
SyncThread2:7 
SyncThread2:8 
SyncThread2:9
```

上面thread1和thread2两个线程因为使用同一个对象执行多线程，此时两个线程拥有相同对象锁（对象锁可以为任意的对象），因此争先抢占同一个同步锁，此时线程是同步执行的，同一时刻只能一个线程执行，谁先抢到锁就谁先循环执行5次。

我们来修改一下调用代码：

```java
Thread thread1 = new Thread(new SyncThread(), "SyncThread1");
Thread thread2 = new Thread(new SyncThread(), "SyncThread2");
thread1.start();
thread2.start();
```

执行结果：

```java
SyncThread1:0 
SyncThread2:1 
SyncThread1:2 
SyncThread2:3 
SyncThread1:4 
SyncThread2:5 
SyncThread2:6 
SyncThread1:7 
SyncThread1:8 
SyncThread2:9
```

此时两个线程锁执行的是不同的两个对象，因此两个线程都拥有各自的锁，这时两个线程就可以异步执行而不会发生堵塞现象。



### synchronized方法

在方法声明中加入 synchronized关键字来声明 synchronized 方法。如：

```java
public synchronized void test(int testVal);
```

相当于在该方法加了把锁，一旦有有线程执行该方法，该线程就会独占这把锁，也就意味着其它线程会被堵在这把锁的外面，无法执行方法内的代码，这时该线程被堵塞，必须等待该线程执行完后，把锁释放了，其它线程才能继续抢占该锁。

```java
/**
 * 同步线程
 */
class SyncThread implements Runnable {
   private static int count;

   public SyncThread() {
      count = 0;
   }

   public synchronized static void method() {
      for (int i = 0; i < 5; i ++) {
         try {
            System.out.println(Thread.currentThread().getName() + ":" + (count++));
            Thread.sleep(100);
         } catch (InterruptedException e) {
            e.printStackTrace();
         }
      }
   }

   public synchronized void run() {
      method();
   }
}
```

调用代码：

```java
SyncThread syncThread1 = new SyncThread();
SyncThread syncThread2 = new SyncThread();
Thread thread1 = new Thread(syncThread1, "SyncThread1");
Thread thread2 = new Thread(syncThread2, "SyncThread2");
thread1.start();
thread2.start();
```

结果：

```java
SyncThread1:0 
SyncThread1:1 
SyncThread1:2 
SyncThread1:3 
SyncThread1:4 
SyncThread2:5 
SyncThread2:6 
SyncThread2:7 
SyncThread2:8 
SyncThread2:9
```

这里会有点“阴险”，虽然线程执行了不同的对像，但却保持了同步执行，原来这是因为run方法调用的是一个静态方法，而静态方法是属于类的，所以syncThread1和syncThread2相当于用了同一把锁。

### synchronized类

```java
/**
 * 同步线程
 */
class SyncThread implements Runnable {
   private static int count;

   public SyncThread() {
      count = 0;
   }

   public static void method() {
      synchronized(SyncThread.class) {
         for (int i = 0; i < 5; i ++) {
            try {
               System.out.println(Thread.currentThread().getName() + ":" + (count++));
               Thread.sleep(100);
            } catch (InterruptedException e) {
               e.printStackTrace();
            }
         }
      }
   }

   public synchronized void run() {
      method();
   }
}
```

效果如synchronized方法的演示一样。



## volatile

volatile主要解决的保持是变量的可见性，说起可见性，那就要从java的内存模型说起：

我们都知道内存的速度是远远跟不上cpu的执行速度的，因此如果cpu每次读取变量都从内存上读取，会大大降低cpu的工作效率，因此java在创建线程并使用变量的时候会从主内存区复制一个副本到私有的高速缓存中，高速缓存比内存那就快多了：

![](https://raw.githubusercontent.com/zchdjb/zchdjb.github.io/master/images/volatile.png)

但是这种机制又会导致变量的不可见性，比如在某一个线程中把s变量复制了一个副本到自己的高速私有缓存中，如果这是该线程修改了s变量的值，但是没有几时把最新的值写会主内存中，就在这时另外一个线程又从主内存中读取s变量的值到自己的私有缓存，这时主内存中的值还是原来的值，也就是说s变量是不可见性的，其它线程修改了它的值，在其它线程是无法看到的，那么这样就会出现脏读了，有可能两个线程执行完，结果还是一样的。

volatile在这种情况就派上用场了，如果s变量被volatile修饰了：

```java
volatile long s = 0L;
```

s变量就拥有了可见性的特征，也就是说s变量在任意线程中被修改了，会立马刷新主内存去s变量的值，使之时刻保持最新的值，**这里有个很重要的特性，就是不仅会立马刷新主内存区的值，还会使其它线程的私有s变量值失效，这样线程就不得不又从主内中再次读取s变量的值，这样，就保持了s变量的可见性**。

**但是！volatile有一个致命的缺点，就是它无法保证变量的原子性**，我们举个栗子：

参考原文地址：http://www.cnblogs.com/aigongsi/archive/2012/04/01/2429166.html

```java
public class Counter {
 
    public static int count = 0;
 
    public static void inc() {
 
        //这里延迟1毫秒，使得结果明显
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
        }
 
        count++;
    }
 
    public static void main(String[] args) {
 
        //同时启动1000个线程，去进行i++计算，看看实际结果
 
        for (int i = 0; i < 1000; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Counter.inc();
                }
            }).start();
        }
 
        //这里每次运行的值都有可能不同,可能为1000
        System.out.println("运行结果:Counter.count=" + Counter.count);
    }
}
```

实际运算结果每次可能都不一样，本机的结果为：运行结果:Counter.count=995，可以看出，在多线程的环境下，Counter.count并没有期望结果是1000。

我们来说说i++运算，它分为取值，加1，赋值三个步骤，当线程1执行到加1步骤时，由于还没有执行赋值改变变量的值，这时候并不会刷新主内存区中的变量，如果此时线程2正好要拷贝该变量的值到自己私有缓存中，问题就出现了，当线程2拷贝完以后，线程1正好执行赋值运算，立马更新主内存区的值，那么此时线程2的副本就是旧的了，脏读又出现了。

因此，想要保持变量的原子性，还是要使用synchronized进行同步执行，尽管synchronized会导致线程阻塞。



