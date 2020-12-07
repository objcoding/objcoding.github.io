---
layout: post
title: "Java并发之AQS源码分析（二）"
categories: Java
tags: concurrent AQS CAS
author: 张乘辉
---

* content
{:toc}
我在[Java并发之AQS源码分析（一）](<https://objcoding.com/2019/05/05/aqs-exclusive-lock/>)这篇文章中，从源码的角度深度剖析了 AQS 独占锁模式下的获取锁与释放锁的逻辑，如果你把这部分搞明白了，再看共享锁的实现原理，思路就会清晰很多。下面我们继续从源码中窥探共享锁的实现原理。











## 共享锁

### 获取锁

```java
public final void acquireShared(int arg) {
  // 尝试获取共享锁，小于0表示获取失败
  if (tryAcquireShared(arg) < 0)
    // 执行获取锁失败的逻辑
    doAcquireShared(arg);
}
```

这里的 tryAcquireShared 方法是留给实现方去实现获取锁的具体逻辑的，我们主要看 doAcquireShared 方法的实现逻辑：

```java
private void doAcquireShared(int arg) {
  // 添加共享锁类型节点到队列中
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      final Node p = node.predecessor();
      if (p == head) {
        // 再次尝试获取共享锁
        int r = tryAcquireShared(arg);
        // 如果在这里成功获取共享锁，会进入共享锁唤醒逻辑
        if (r >= 0) {
          // 共享锁唤醒逻辑
          setHeadAndPropagate(node, r);
          p.next = null; // help GC
          if (interrupted)
            selfInterrupt();
          failed = false;
          return;
        }
      }
      // 与独占锁相同的挂起逻辑
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

看到上面的代码，是不是有一种熟悉的感觉，**同样是采用了自旋机制，在线程挂起之前，不断地循环尝试获取锁，不同的是，一旦获取共享锁，会调用 setHeadAndPropagate 方法同时唤醒后继节点，实现共享模式**，下面是唤醒后继节点代码逻辑：

```java
private void setHeadAndPropagate(Node node, int propagate) {
  // 头节点
  Node h = head; 
  // 设置当前节点为新的头节点
  // 这里不需要加锁操作，因为获取共享锁后，会从FIFO队列中依次唤醒队列，并不会产生并发安全问题
  setHead(node);
  if (propagate > 0 || h == null || h.waitStatus < 0 ||
      (h = head) == null || h.waitStatus < 0) {
    // 后继节点
    Node s = node.next;
    // 如果后继节点为空或者后继节点为共享类型，则进行唤醒后继节点
    // 这里后继节点为空意思是只剩下当前头节点了
    if (s == null || s.isShared())
      doReleaseShared();
  }
}
```

该方法主要做了两个重要的步骤：

1. **将当前节点设置为新的头节点，这点很重要，这意味着当前节点的前置节点（旧头节点）已经获取共享锁了，从队列中去除；**
2. **调用 doReleaseShared 方法，它会调用 unparkSuccessor 方法唤醒后继节点。**



### 释放锁

```java
public final boolean releaseShared(int arg) {
  // 由用户自行实现释放锁条件
  if (tryReleaseShared(arg)) {
    // 执行释放锁
    doReleaseShared();
    return true;
  }
  return false;
}
```

下面是释放锁逻辑：

```java
private void doReleaseShared() {
  for (;;) {
    // 从头节点开始执行唤醒操作
    // 这里需要注意，如果从setHeadAndPropagate方法调用该方法，那么这里的head是新的头节点
    Node h = head;
    if (h != null && h != tail) {
      int ws = h.waitStatus;
      //表示后继节点需要被唤醒
      if (ws == Node.SIGNAL) {
        // 初始化节点状态
        //这里需要CAS原子操作，因为setHeadAndPropagate和releaseShared这两个方法都会顶用doReleaseShared，避免多次unpark唤醒操作
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
          // 如果初始化节点状态失败，继续循环执行
          continue;            // loop to recheck cases
        // 执行唤醒操作
        unparkSuccessor(h);
      }
      //如果后继节点暂时不需要唤醒，那么当前头节点状态更新为PROPAGATE，确保后续可以传递给后继节点
      else if (ws == 0 &&
               !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        continue;                // loop on failed CAS
    }
    // 如果在唤醒的过程中头节点没有更改，退出循环
    // 这里防止其它线程又设置了头节点，说明其它线程获取了共享锁，会继续循环操作
    if (h == head)                   // loop if head changed
      break;
  }
}
```

共享锁的释放锁逻辑比独占锁的释放锁逻辑稍微复杂，原因是共享锁需要释放队列中所有共享类型的节点，因此需要循环操作，由于释放锁过程中会涉及多个地方修改节点状态，此时需要 CAS 原子操作来并发安全。



获取共享锁流程图：

![](https://gitee.com/objcoding/md-picture/raw/master/img/aqs_2.jpg)





## 总结

跟独占锁相比，从流程图也可看出，共享锁的主要特征是当有一个线程获取到锁之后，那么它就会依次唤醒等待队列中可以跟它共享的节点，当然这些节点也是共享锁类型。








