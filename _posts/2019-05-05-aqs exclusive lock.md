---
layout: post
title: "Java并发之AQS源码分析（一）"
categories: Java
tags: concurrent AQS CAS
author: zch
---

* content
{:toc}
AQS 全称是 AbstractQueuedSynchronizer，顾名思义，是一个用来构建锁和同步器的框架，它底层用了 CAS 技术来保证操作的原子性，同时利用 FIFO 队列实现线程间的锁竞争，将基础的同步相关抽象细节放在 AQS，这也是 ReentrantLock、CountDownLatch 等同步工具实现同步的底层实现机制。它能够成为实现大部分同步需求的基础，也是 J.U.C 并发包同步的核心基础组件。











## AQS 结构剖析

AQS 就是建立在 CAS 的基础之上，增加了大量的实现细节，例如获取同步状态、FIFO 同步队列，独占式锁和共享式锁的获取和释放等等，这些都是 AQS 类对于同步操作抽离出来的一些通用方法，这么做也是为了对实现的一个同步类屏蔽了大量的细节，大大降低了实现同步工具的工作量，这也是为什么 AQS 是其它许多同步类的基类的原因。

现在我们来直接定位到类 java.util.concurrent.locks.AbstractQueuedSynchronizer，下面是 AQS 类的几个重要字段与方法列出来：

```java
public abstract class AbstractQueuedSynchronizer
  extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    private transient volatile Node head;

    private transient volatile Node tail;

    private volatile int state;

    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

    // ...

 }
```

1. head 字段为等待队列的头节点，表示当前正在执行的节点；
2. tail 字段为等待队列的尾节点；
3. state 字段为同步状态，其中 state > 0 为有锁状态，每次加锁就在原有 state 基础上加 1，即代表当前持有锁的线程加了 state 次锁，反之解锁时每次减一，当 statte = 0 为无锁状态；
4. 通过 compareAndSetState 方法操作 CAS 更改 state 状态，保证 state 的原子性。

有没有发现，这几个字段都用 volatile 关键字进行修饰，以确保多线程间保证字段的可见性。

AQS 提供了两种锁，分别是独占锁和共享锁，独占锁指的是操作被认作一种独占操作，比如 ReentrantLock，它实现了独占锁的方法，而共享锁则指的是一个非独占操作，比如一些同步工具 CountDownLatch 和 Semaphore 等同步工具，下面是 AQS 对这两种锁提供的抽象方法。

独占锁：

```java
// 获取锁方法
protected boolean tryAcquire(int arg) {
  throw new UnsupportedOperationException();
}
// 释放锁方法
protected boolean tryRelease(int arg) {
  throw new UnsupportedOperationException();
}
    
```

共享锁：

```java
// 获取锁方法
protected int tryAcquireShared(int arg) {
  throw new UnsupportedOperationException();
}
// 释放锁方法
protected boolean tryReleaseShared(int arg) {
  throw new UnsupportedOperationException();
}
```

在我们平时开发中，基本不用直接使用 AQS，我们平时都是直接使用 JDK 自带的同步类工具，如 ReentrantLock、CountDownLatch 和 Semaphore 等，它们已经可以满足绝大部分的需求了，后面会抽几篇文章单独讲一下这些同步类工具是如何使用 AQS 的，这对于我们如何构建自定义的同步工具，有很大的帮助。

下面是同步队列节点的结构：

![aqs node](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/aqs.png)

用大神的注释来形象地描述一下队列的模型：

```java
/**
  * <pre>
  *      +------+  prev +-----+       +-----+
  * head |      | <---- |     | <---- |     |  tail
  *      +------+       +-----+       +-----+
  * </pre>
  */
```

这是一个普通双向链表的节点结构，多了 thread 字段用于存储当前线程对象，同时每个节点都有一个 waitStatus 等待状态，一共有四种状态：

1. CANCELLED（1）：取消状态，如果当前线程的前置节点状态为 CANCELLED，则表明前置节点已经等待超时或者已经被中断了，这时需要将其从等待队列中删除。
2. SIGNAL（-1）：等待触发状态，如果当前线程的前置节点状态为 SIGNAL，则表明当前线程需要阻塞。
3. CONDITION（-2）：等待条件状态，表示当前节点在等待 condition，即在 condition 队列中。
4. PROPAGATE（-3）：状态需要向后传播，表示 releaseShared 需要被传播给后续节点，仅在共享锁模式下使用。


可以这么理解：head 节点可以表示成当前持有锁的线程的节点，其余线程竞争锁失败后，会加入到队尾，tail 始终指向队列的最后一个节点。

AQS 的结构大概可总结为以下 3 部分：

1. **用 volatile 修饰的整数类型的 state 状态，用于表示同步状态，提供 getState 和 setState 来操作同步状态；**
2. **提供了一个 FIFO 等待队列，实现线程间的竞争和等待，这是 AQS 的核心；**
3. **AQS 内部提供了各种基于 CAS 原子操作方法，如 compareAndSetState 方法，并且提供了锁操作的acquire和release方法。**






## 独占锁

**独占锁的原理是如果有线程获取到锁，那么其它线程只能是获取锁失败，然后进入等待队列中等待被唤醒。**

### 获取锁

获取独占锁方法：

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

源码解读：

1. **通过 tryAcquire(arg) 方法尝试获取锁，这个方法需要实现类自己实现获取锁的逻辑，获取锁成功后则不执行后面加入等待队列的逻辑了；**
2. **如果尝试获取锁失败后，则执行 addWaiter(Node.EXCLUSIVE) 方法将当前线程封装成一个 Node 节点对象，并加入队列尾部；**
3. **把当前线程执行封装成 Node 节点后，继续执行 acquireQueued 的逻辑，该逻辑主要是判断当前节点的前置节点是否是头节点，来尝试获取锁，如果获取锁成功，则当前节点就会成为新的头节点，这也是获取锁的核心逻辑。**



基于上面源码的步骤分析后，我们一步步往下看源码具体实现：

```java
private Node addWaiter(Node mode) {
  // 创建一个基于当前线程的节点，该节点是 Node.EXCLUSIVE 独占式类型
  Node node = new Node(Thread.currentThread(), mode);
  // Try the fast path of enq; backup to full enq on failure
  Node pred = tail;
  // 这里先判断队尾是否为空，如果不为空则直接将节点加入队尾
  if (pred != null) {
    node.prev = pred;
    // 采取 CAS 操作，将当前节点设置为队尾节点，由于采用了 CAS 原子操作，无论并发怎么修改，都有且只有一条线程可以修改成功，其余都将执行后面的enq方法
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  enq(node);
  return node;
}
```

简单来说  addWaiter(Node mode) 方法做了以下事情：

1. 创建基于当前线程的独占式类型的节点；
2. 利用 CAS 原子操作，将节点加入队尾。

我们继续看 enq(Node node) 方法：

```java
private Node enq(final Node node) {
  // 自旋操作
  for (;;) {
    Node t = tail;
    // 如果队尾节点为空，那么进行CAS操作初始化队列
    if (t == null) {
      // 这里很关键，即如果队列为空，那么此时必须初始化队列，初始化一个空的节点表示队列头，用于表示当前正在执行的节点，头节点即表示当前正在运行的节点
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      node.prev = t;
      // 这一步也是采取CAS操作，将当前节点加入队尾，如果失败的话，自旋继续修改直到成功为止
      if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
      }
    }
  }
}
```

 enq(final Node node) 方法主要做了以下事情：

1. 采用自旋机制，这是 aqs 里面很重要的一个机制；
2. **如果队尾节点为空，则初始化队列，将头节点设置为空节点，头节点即表示当前正在运行的节点；**
3. 如果队尾节点不为空，则继续采取 CAS 操作，将当前节点加入队尾，不成功则继续自旋，直到成功为止；

对比了上面两段代码，不难看出，首先是判断队尾是否为空，先进行一次 CAS 入队操作，如果失败则进入 enq(final Node node) 方法执行完整的入队操作。

完整的入队操作简单来说就是：**如果队列为空，初始化队列，并将头节点设为空节点，表示当前正在运行的节点，然后再将当前线程的节点加入到队列尾部。**

关于队列的初始化与入队，务必理解透彻。



经过上面 CAS 不断尝试，这时当前节点已经成功加入到队尾了，接下来就到了acquireQueued 的逻辑，我们继续往下看源码：

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    // 线程中断标记字段
    boolean interrupted = false;
    for (;;) {
      // 获取当前节点的 pred 节点
      final Node p = node.predecessor();
      // 如果 pred 节点为 head 节点，那么再次尝试获取锁
      if (p == head && tryAcquire(arg)) {
        // 获取锁之后，那么当前节点也就成为了 head 节点
        setHead(node);
        p.next = null; // help GC
        failed = false;
        // 不需要挂起，返回 false
        return interrupted;
      }
      // 获取锁失败，则进入挂起逻辑
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

这一步 acquireQueued(final Node node, int arg) 方法主要做了以下事情：

1. 判断当前节点的 pred 节点是否为 head 节点，如果是，则尝试获取锁；
2. 获取锁失败后，进入挂起逻辑。


提醒一点：**我们上面也说过，head 节点代表当前持有锁的线程，那么如果当前节点的 pred 节点是 head 节点，很可能此时 head 节点已经释放锁了，所以此时需要再次尝试获取锁。**

接下来继续看挂起逻辑源码：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    // 如果 pred 节点为 SIGNAL 状态，返回true，说明当前节点需要挂起
    return true;
  // 如果ws > 0,说明节点状态为CANCELLED，需要从队列中删除
  if (ws > 0) {
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    // 如果是其它状态，则操作CAS统一改成SIGNAL状态
    // 由于这里waitStatus的值只能是0或者PROPAGATE，所以我们将节点设置为SIGNAL，从新循环一次判断
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```

这一步 shouldParkAfterFailedAcquire(Node pred, Node node) 方法主要做了以下事情：

1. 判断 pred 节点状态，如果为 SIGNAL 状态，则直接返回 true 执行挂起；
2. 删除状态为 CANCELLED 的节点；
3. 若 pred 节点状态为 0 或者 PROPAGATE，则将其设置为为 SIGNAL，再从 acquireQueued 方法自旋操作从新循环一次判断。

**通俗来说就是：根据 pred 节点状态来判断当前节点是否可以挂起，如果该方法返回 false，那么挂起条件还没准备好，就会重新进入 acquireQueued(final Node node, int arg) 的自旋体，重新进行判断。如果返回 true，那就说明当前线程可以进行挂起操作了，那么就会继续执行挂起。**

这里需要注意的时候，节点的初始值为 0，因此如果获取锁失败，会尝试将节点设置为 SIGNAL。

继续看挂起逻辑：

```java
private final boolean parkAndCheckInterrupt() {
  LockSupport.park(this);
  return Thread.interrupted();
}
```

LockSupport 是用来创建锁和其他同步类的基本**线程阻塞**原语。LockSupport 提供 park() 和 unpark() 方法实现阻塞线程和解除线程阻塞。release 释放锁方法逻辑会调用 LockSupport.unPark 方法来唤醒后继节点。



获取独占锁流程图：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/aqs.jpg)





### 释放锁

释放锁方法：

```java
public final boolean release(int arg) {
  if (tryRelease(arg)) {
    Node h = head;
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```

释放锁的方法源码就很好理解，通过 tryRelease(arg) 方法尝试释放锁，这个方法需要实现类自己实现释放锁的逻辑，释放锁成功后则执行后面的唤醒后续节点的逻辑了，然后判断 head 节点不为空并且 head 节点状态不为 0，因为 addWaiter 方法默认的节点状态为 0，此时节点还没有进入就绪状态。

继续往下看源码：

```java
private void unparkSuccessor(Node node) {
  int ws = node.waitStatus;
  if (ws < 0)
    // 将头节点的状态设置为0
    // 这里会尝试清除头节点的状态，改为初始状态
    compareAndSetWaitStatus(node, ws, 0);
  
  // 后继节点
  Node s = node.next;
  // 如果后继节点为null，或者已经被取消了
  if (s == null || s.waitStatus > 0) {
    s = null;
    // for循环从队列尾部一直往前找可以唤醒的节点
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  if (s != null)
    // 唤醒后继节点
    LockSupport.unpark(s.thread);
}
```

从源码可看出：**释放锁主要是将头节点的后继节点唤醒，如果后继节点不符合唤醒条件，则从队尾一直往前找，直到找到符合条件的节点为止**。





## 总结

这篇文章主要讲述了 AQS 的内部结构和它的同步实现原理，并从源码的角度深度剖析了 AQS 独占锁模式下的获取锁与释放锁的逻辑，并且从源码中我们得出：**在独占锁模式下，用 state 值表示锁并且 0 表示无锁状态，0 -> 1 表示从无锁到有锁，仅允许一条线程持有锁，其余的线程会被包装成一个 Node 节点放到队列中进行挂起，队列中的头节点表示当前正在执行的线程，当头节点释放后会唤醒后继节点，从而印证了 AQS 的队列是一个 FIFO 同步队列。**














