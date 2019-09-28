---
layout: post
title: "RocketMQ主从如何同步消息消费进度？"
categories: RocketMQ
tags: HA Master Slave 
author: zch
---

* content
{:toc}
前面我也跟大家讲述了 RocketMQ 读写分离的规则，但是你可能会问，主从服务器之间的消费进度是如何保持同步的？下面我来给大家解答一下。









如果消费者消费模式不同，也会有不同的保存方式，消费者端的消息消费进度保存到 OffsetStore 中，他有两个实现类：

```java
org.apache.rocketmq.client.consumer.store.LocalFileOffsetStore // 本地消费进度保存实现
org.apache.rocketmq.client.consumer.store.RemoteBrokerOffsetStore // 远程消费进度保存实现
```

其中，如果是广播模式消费，消息的消费进度是保存到本地，如果是集群消费模式，消息的消费进度则是保存到 Broker，但无论是保存到本地，还是保存到 Broker，消费者都会在本地留一份缓存，我们暂且看看集群消费模式下，消息消费进度的缓存是如何保存的：

org.apache.rocketmq.client.consumer.store.RemoteBrokerOffsetStore#updateOffset:

```java
public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly) {
  if (mq != null) {
    AtomicLong offsetOld = this.offsetTable.get(mq);
    if (null == offsetOld) {
      offsetOld = this.offsetTable.putIfAbsent(mq, new AtomicLong(offset));
    }

    if (null != offsetOld) {
      if (increaseOnly) {
        MixAll.compareAndIncreaseOnly(offsetOld, offset);
      } else {
        offsetOld.set(offset);
      }
    }
  }
}
```

消息者在消费完消息后，会调用以上方法，讲消费进度放入 offsetTable 缓存中，当 Rebalance 负载重新分配生成 PullRequest 对象时，会调用 RemoteBrokerOffsetStore.readOffset 方法从 offsetTable 缓存中取出对应的消费进度缓存值，再将该值放进 PullRequest 对象中，接下来消息拉取时就很将消息消费进度缓存发送到 Broker 端，所以我们继续看 Broker 端的处理逻辑。

之前整理 Broker 启动流程时，发现 Broker 启动时会开启一个定时任务：

org.apache.rocketmq.broker.BrokerController#initialize:

```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            BrokerController.this.slaveSynchronize.syncAll();
        } catch (Throwable e) {
            log.error("ScheduledTask syncAll slave exception", e);
        }
    }
}, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
```

如果 Broker 是从服务器，则会开启以上定时任务。

org.apache.rocketmq.broker.slave.SlaveSynchronize#syncAll:

```java
public void syncAll() {
  this.syncTopicConfig();
  this.syncConsumerOffset();
  this.syncDelayOffset();
  this.syncSubscriptionGroupConfig();
}
```

在主服务器没有宕机的情况下，从服务器会定时从主服务器中同步消息消费进度等信息，那现在问题来了，由于这个同步是单方面同步，即只会从服务器同步主服务器，那如果主服务器宕机了之后，消费者切换成从服务器拉取消息进行消费，如果之后主服务器启动了，从服务器在把已经消费过的偏移量同步过来，那岂不是造成同步消费了？

其实消费者取在拉取消息的时候，如果消费者的缓存中存在消费进度，也会向 Broker 更新消息消费进度，所以即使是主服务器挂了，在它重新启动之后，消费者的消费进度没有丢失，依然会更新主服务器的消息消费进度，这样一来，消费端与主服务器只挂了器中一个，并不会导致消息重新被消费，具体代码逻辑如下：

org.apache.rocketmq.broker.processor.PullMessageProcessor#processRequest：

```java
boolean storeOffsetEnable = brokerAllowSuspend;
storeOffsetEnable = storeOffsetEnable && hasCommitOffsetFlag;
storeOffsetEnable = storeOffsetEnable
    && this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE;
if (storeOffsetEnable) {
 this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(channel), requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId(), requestHeader.getCommitOffset());
}
```

其中 brokerAllowSuspend 表示 broker 是否允许挂起，该值默认为 true，hasCommitOffsetFlag 表示息消费者在内存中是否缓存了消息消费进度，从代码逻辑可看出，如果 Broker 为主服务器，并且 brokerAllowSuspend 和 hasCommitOffsetFlag 都为true，那么就会将消费者消费进度更新到本地。

