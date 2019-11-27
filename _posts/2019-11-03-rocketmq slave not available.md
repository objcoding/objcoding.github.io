---
layout: post
title: "RocketMQ 同步复制 SLAVE_NOT_AVAILABLE 异常源码分析"
categories: RocketMQ
tags: 
author: zch
---

* content
{:toc}
最近在 RocketMQ 钉钉官方群中看到有人反馈说 broker 主从部署，在发布消息的时候会报 SLAVE_NOT_AVAILABLE 异常，报这个异常的前提 master 的模式一定为 SYNC_MASTER（同步复制），从 异常码可以直接判断的一种原因就是因为 slave 挂掉了，导致  slave 不可用，但是他说 slave 一切正常。

于是我决定撸一波源码。





既然是主从同步的问题，那么我们直接定位到处理同步复制的方法：

org.apache.rocketmq.store.CommitLog#handleHA

```java
public void handleHA(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
  if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
    HAService service = this.defaultMessageStore.getHaService();
    if (messageExt.isWaitStoreMsgOK()) {
      // Determine whether to wait
      if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
        GroupCommitRequest  request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
        service.putRequest(request);
        service.getWaitNotifyObject().wakeupAll();
        boolean flushOK =
          request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
        if (!flushOK) {
          log.error("do sync transfer other node, wait return, but failed, topic: " + messageExt.getTopic() + " tags: "
                    + messageExt.getTags() + " client address: " + messageExt.getBornHostNameString());
          putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
        }
      }
      // Slave problem
      else {
        // Tell the producer, slave not available
        putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
      }
    }
  }
}
```

消息写入时需要判断 master 是否为 SYNC_MASTER 模式，从源码可以看出来，isSlaveOK() 方法决定是否报 SLAVE_NOT_AVAILABLE 异常码的关键逻辑，所以关键就是要看这个方法：

org.apache.rocketmq.store.ha.HAService#isSlaveOK

```java
public boolean isSlaveOK(final long masterPutWhere) {
    boolean result = this.connectionCount.get() > 0;
    result =
        result
            && ((masterPutWhere - this.push2SlaveMaxOffset.get()) < this.defaultMessageStore
            .getMessageStoreConfig().getHaSlaveFallbehindMax());
    return result;
}
```

从源码的逻辑看，masterPutWhere = result.getWroteOffset() + result.getWroteBytes()，其中 wroteOffset 表示从那个位移开始写入，wroteBytes 表示写入的消息量，因此 masterPutWhere 表示 master 最大的消息拉取位移，push2SlaveMaxOffset 表示的是此时 slave 拉取最大的位移，haSlaveFallbehindMax 表示 slave 主从同步同步复制时最多可落后 master 的位移，masterPutWhere - this.push2SlaveMaxOffset.get() 即可表示此时 slave 落后 master 的位移量，如果大于 haSlaveFallbehindMax，则报 SLAVE_NOT_AVAILABLE 给客户端，不过不用担心，只要 slave 没有挂掉，slave 的同步位移肯定能够追上来。

push2SlaveMaxOffset 参数值 是 slave 与 master 保持一个心跳频率，定时上报给 master，master 再根据这个值判断 slave 落后 master 多少位移量。

下面重点分析 slave 如何上报 push2SlaveMaxOffset 给master。

master 收到 slave 的位移量之后，是从以下方法进行更新的：

org.apache.rocketmq.store.ha.HAService#notifyTransferSome

```java
public void notifyTransferSome(final long offset) {
    for (long value = this.push2SlaveMaxOffset.get(); offset > value; ) {
        boolean ok = this.push2SlaveMaxOffset.compareAndSet(value, offset);
        if (ok) {
            this.groupTransferService.notifyTransferSome();
            break;
        } else {
            value = this.push2SlaveMaxOffset.get();
        }
    }
}
```

从调用栈来看，该方法在服务端处理读请求类中调用了，我们接着往下看：

org.apache.rocketmq.store.ha.HAConnection.ReadSocketService#processReadEvent

```java
if (readSize > 0) {
  readSizeZeroTimes = 0;
  this.lastReadTimestamp = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
  if ((this.byteBufferRead.position() - this.processPostion) >= 8) {
    int pos = this.byteBufferRead.position() - (this.byteBufferRead.position() % 8);
    long readOffset = this.byteBufferRead.getLong(pos - 8);
    this.processPostion = pos;

    HAConnection.this.slaveAckOffset = readOffset;
    if (HAConnection.this.slaveRequestOffset < 0) {
      HAConnection.this.slaveRequestOffset = readOffset;
      log.info("slave[" + HAConnection.this.clientAddr + "] request offset " + readOffset);
    }

    HAConnection.this.haService.notifyTransferSome(HAConnection.this.slaveAckOffset);
  }
```

如上源码逻辑，如果读取到的字节大于 0，并且大于等于 8，则说明了收到了 slave 端反馈过来的位移量，于是将其取出并更新到  push2SlaveMaxOffset 中。

接着我们来看 slave 是如何上报位移的。

org.apache.rocketmq.store.ha.HAService.HAClient#run

```java
if (this.isTimeToReportOffset()) {
  boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset);
  if (!result) {
    this.closeMaster();
  }
}
```

以上逻辑在 slave 端处理拉取同步消息线程 run 方法中，首先判断是否到了需要上报位移的时间间隔了，到了直接调用上报位移方法。

org.apache.rocketmq.store.ha.HAService.HAClient#isTimeToReportOffset

```java
private boolean isTimeToReportOffset() {
    long interval =
        HAService.this.defaultMessageStore.getSystemClock().now() - this.lastWriteTimestamp;
    boolean needHeart = interval > HAService.this.defaultMessageStore.getMessageStoreConfig()
        .getHaSendHeartbeatInterval();

    return needHeart;
}
```

首先求出距离上次同步消息的时时间间隔的大小，再与 haSendHeartbeatInterval 作对比，若大于 haSendHeartbeatInterval 则发送心跳包上报位移。

org.apache.rocketmq.store.ha.HAService.HAClient#reportSlaveMaxOffset

```java
private boolean reportSlaveMaxOffset(final long maxOffset) {
    this.reportOffset.position(0);
    this.reportOffset.limit(8);
    this.reportOffset.putLong(maxOffset);
    this.reportOffset.position(0);
    this.reportOffset.limit(8);

    for (int i = 0; i < 3 && this.reportOffset.hasRemaining(); i++) {
        try {
            this.socketChannel.write(this.reportOffset);
        } catch (IOException e) {
            log.error(this.getServiceName()
                + "reportSlaveMaxOffset this.socketChannel.write exception", e);
            return false;
        }
    }

    return !this.reportOffset.hasRemaining();
}
```

该方法向主服务器上报已拉取的位移，具体做法是将 ByteBuffer 读取位置 position 值为 0，其实调用 flip() 方法也可以，然后调用 putLong() 方法将 maxOffset 写入 ByteBuffer，将 limit 设置为 8，跟写入 ByteBuffer 中的 maxOffset（long 型）大小一样，最后采取 for 循环将 maxOffset 写入网络通道中，并调用 hasRemaining() 方法，该方法的逻辑为判断 position 是否小于 limit，即判断 ByteBuffer 中的字节流是否全部写入到通道中。

最后总结，如果 slave 正常运行，报这个错是正常的，你可以适当调整 haSendHeartbeatInterval 参数（1000 * 5）的大小，它决定 slave 上报同步位移的心跳频率，以及 haSlaveFallbehindMax 参数值（默认 1024 * 1024 * 256），它决定允许 slave 最多落后 master 的位移。