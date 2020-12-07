---
layout: post
title: "深度剖析 Kafka Producer 的缓冲池机制【图解 + 源码分析】"
categories: Kakfa
tags: 缓冲池 生产者
author: 张乘辉
---

* content
{:toc}
上次跟大家分享的文章「[Kafka Producer 异步发送消息居然也会阻塞？](https://mp.weixin.qq.com/s/wbTIW3MkCxaCb8ToFe5wiA)」中提到了缓冲池，后面再经过一番阅读源码后，发现了这个缓冲池设计的很棒，被它的设计思想优雅到了，所以忍不住跟大家继续分享一波。













在新版的 Kafka Producer 中，设计了一个消息缓冲池，在创建 Producer 时会默认创建一个大小为 32M 的缓冲池，也可以通过 buffer.memory 参数指定缓冲池的大小，同时缓冲池被切分成多个内存块，内存块的大小就是我们创建 Producer 时传的 batch.size 大小，默认大小 16384，而每个 Batch 都会包含一个 batch.size 大小的内存块，消息就是存放在内存块当中。整个缓冲池的结构如下图所示：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200913211915.png)

客户端将消息追加到对应主题分区的某个 Batch 中，如果 Batch 已经满了，则会新建一个 Batch，同时向缓冲池（RecordAccumulator）申请一块大小为 batch.size 的内存块用于存储消息。

当 Batch 的消息被发到了 Broker 后，Kafka Producer 就会移除该 Batch，既然 Batch 持有某个内存块，那必然就会涉及到 GC 问题，如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200913191436.png)

以上，频繁的申请内存，用完后就丢弃，必然导致频繁的 GC，造成严重的性能问题。那么，Kafka 是怎么做到避免频繁 GC 的呢？

前面说过了，缓冲池在设计逻辑上面被切分成一个个大小相等的内存块，当消息发送完毕，归还给缓冲池不就可以避免被回收了吗？

缓冲池的内存持有类是 BufferPool，我们先来看下 BufferPool 都有哪些成员：

```java
public class BufferPool {
  // 总的内存大小
  private final long totalMemory;
  // 每个内存块大小，即 batch.size
  private final int poolableSize;
  // 申请、归还内存的方法的同步锁
  private final ReentrantLock lock;
  // 空闲的内存块
  private final Deque<ByteBuffer> free;
  // 需要等待空闲内存块的事件
  private final Deque<Condition> waiters;
  /** Total available memory is the sum of nonPooledAvailableMemory and the number of byte buffers in free * poolableSize.  */
  // 缓冲池还未分配的空闲内存，新申请的内存块就是从这里获取内存值
  private long nonPooledAvailableMemory;
	// ...
}
```

从 BufferPool 的成员可看出，缓冲池实际上由一个个 ByteBuffer 组成的，BufferPool 持有这些内存块，并保存在成员 free 中，free 的总大小由 totalMemory 作限制，而 nonPooledAvailableMemory 则表示还剩下缓冲池还剩下多少内存还未被分配。

当 Batch 的消息发送完毕后，就会将它持有的内存块归还到 free 中，以便后面的 Batch 申请内存块时不再创建新的 ByteBuffer，从 free 中取就可以了，从而避免了内存块被 JVM 回收的问题。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200913212301.png)



接下来跟大家一起分析申请内存和归还内存是如何实现的。

1、申请内存

申请内存的入口：

`org.apache.kafka.clients.producer.internals.BufferPool#allocate`

1）内存足够的情况

当用户请求申请内存时，如果发现 free 中有空闲的内存，则直接从中取：

```java
if (size == poolableSize && !this.free.isEmpty()){
  return this.free.pollFirst(); 
}
```

这里的 size 即申请的内存大小，它等于 `Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));`

即如果你的消息大小小于 batchSize，则申请的内存大小为 batchSize，那么上面的逻辑就是如果申请的内存大小等于 batchSize 并且 free 不空闲，则直接从 free 中获取。

我们不妨想一下，为什么 Kafka 一定要申请内存大小等于 batchSize，才能从 free 获取空闲的内存块呢？

前面也说过，缓冲池的内存块大小是固定的，它等于 batchSize，如果申请的内存比 batchSize 还大，说明一条消息所需要存放的内存空间比内存块的内存空间还要大，因此不满足需求，不满组需求怎么办呢？我们接着往下分析：

```java
// now check if the request is immediately satisfiable with the
// memory on hand or if we need to block
int freeListSize = freeSize() * this.poolableSize;
if (this.nonPooledAvailableMemory + freeListSize >= size) {
  // we have enough unallocated or pooled memory to immediately
  // satisfy the request, but need to allocate the buffer
  freeUp(size);
  this.nonPooledAvailableMemory -= size;
}
```

freeListSize：指的是 free 中已经分配好并且已经回收的空闲内存块总大小；

nonPooledAvailableMemory：缓冲池还未分配的空闲内存，新申请的内存块就是从这里获取内存值；

this.nonPooledAvailableMemory + freeListSize：即缓冲池中总的空闲内存空间。

如果缓冲池的内存空间比申请内存大小要大，则调用  `freeUp(size);` 方法，接着将空闲的内存大小减去申请的内存大小。

```java
private void freeUp(int size) {
  while (!this.free.isEmpty() && this.nonPooledAvailableMemory < size)
    this.nonPooledAvailableMemory += this.free.pollLast().capacity();
}
```

freeUp 这个方法很有趣，它的思想是这样的：

如果未分配的内存大小比申请的内存还要小，那只能从已分配的内存列表 free 中将内存空间要回来，直到 nonPooledAvailableMemory 比申请内存大为止。

2）内存不足的情况

在我的「[Kafka Producer 异步发送消息居然也会阻塞？](https://mp.weixin.qq.com/s/wbTIW3MkCxaCb8ToFe5wiA)」这篇文章当中也提到了，当缓冲池的内存块用完后，消息追加调用将会被阻塞，直到有空闲的内存块。

阻塞等待的逻辑是怎么实现的呢？

```java
// we are out of memory and will have to block
int accumulated = 0;
Condition moreMemory = this.lock.newCondition();
try {
  long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
  this.waiters.addLast(moreMemory);
  // loop over and over until we have a buffer or have reserved
  // enough memory to allocate one
  while (accumulated < size) {
    long startWaitNs = time.nanoseconds();
    long timeNs;
    boolean waitingTimeElapsed;
    try {
      waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
    } finally {
      long endWaitNs = time.nanoseconds();
      timeNs = Math.max(0L, endWaitNs - startWaitNs);
      recordWaitTime(timeNs);
    }

    if (waitingTimeElapsed) {
      throw new TimeoutException("Failed to allocate memory within the configured max blocking time " + maxTimeToBlockMs + " ms.");
    }

    remainingTimeToBlockNs -= timeNs;

    // check if we can satisfy this request from the free list,
    // otherwise allocate memory
    if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
      // just grab a buffer from the free list
      buffer = this.free.pollFirst();
      accumulated = size;
    } else {
      // we'll need to allocate memory, but we may only get
      // part of what we need on this iteration
      freeUp(size - accumulated);
      int got = (int) Math.min(size - accumulated, this.nonPooledAvailableMemory);
      this.nonPooledAvailableMemory -= got;
      accumulated += got;
    }
  }
```

以上源码的大致逻辑：

首先创建一个本次等待 Condition，并且把它添加到类型为 Deque 的 waiters 中（后面在归还内存中会唤醒），while 循环不断收集空闲的内存，直到内存比申请内存大时退出，在 while 循环过程中，调用 Condition#await 方法进行阻塞等待，归还内存时会被唤醒，唤醒后会判断当前申请内存是否大于 batchSize，如果等与 batchSize 则直接将归还的内存返回即可，如果当前申请的内存大于 大于 batchSize，则需要调用 freeUp 方法从 free 中释放空闲的内存出来，然后进行累加，直到大于申请的内存为止。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200913220918.png)



2、归还内存

申请内存的入口：

`org.apache.kafka.clients.producer.internals.BufferPool#deallocate(java.nio.ByteBuffer, int)`

```java
public void deallocate(ByteBuffer buffer, int size) {
  lock.lock();
  try {
    if (size == this.poolableSize && size == buffer.capacity()) {
      buffer.clear();
      this.free.add(buffer);
    } else {
      this.nonPooledAvailableMemory += size;
    }
    Condition moreMem = this.waiters.peekFirst();
    if (moreMem != null)
      moreMem.signal();
  } finally {
    lock.unlock();
  }
}
```

归还内存块的逻辑比较简单：

如果归还的内存块大小等于 batchSize，则将其清空后添加到缓冲池的 free 中，即将其归还给缓冲池，避免了 JVM GC 回收该内存块。如果不等于呢？直接将内存大小累加到未分配并且空闲的内存大小值中即可，内存就无需归还了，等待 JVM GC 回收掉，最后唤醒正在等待空闲内存的线程。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200913221545.png)



经过以上的源码分析之后，给大家指出需要注意的一个问题，如果设置不当，会给 Producer 端带来严重的性能影响：

如果你的消息大小比 batchSize 还要大，则不会从 free 中循环获取已分配好的内存块，而是重新创建一个新的 ByteBuffer，并且该 ByteBuffer 不会被归还到缓冲池中（JVM GC 回收），如果此时 nonPooledAvailableMemory 比消息体还要小，还会将 free 中空闲的内存块销毁（JVM GC 回收），以便缓冲池中有足够的内存空间提供给用户申请，这些动作都会导致频繁 GC 的问题出现。

因此，需要根据业务消息的大小，适当调整 batch.size 的大小，避免频繁 GC。







