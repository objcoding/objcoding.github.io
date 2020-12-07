---
layout: post
title: "RocketMQ主从读写分离机制"
categories: RocketMQ
tags: HA Master Slave 
author: 张乘辉
---

* content
{:toc}
一般来说，选择主从备份实现高可用的架构中，都会具备读写分离机制，比如 MySql 读写分离，客户端可以向主从服务器读取数据，但客户写数据只能通过主服务器。

RocketMQ 的读写分离机制又跟上述描写的不太一致，RocketMQ 有属于自己的一套读写分离逻辑，它会判断主服务器的消息堆积量来决定消费者是否向从服务器拉取消息消费。









决定消费者是否向从服务器拉取消息消费的值存在 GetMessageResult 类中：

org.apache.rocketmq.store.GetMessageResult：

```java
private boolean suggestPullingFromSlave = false;
```

其默认值为 false，即默认消费者不会消费从服务器，以下逻辑可以改变该值：

org.apache.rocketmq.store.DefaultMessageStore#getMessage：

```java
long diff = maxOffsetPy - maxPhyOffsetPulling;
long memory = (long) (StoreUtil.TOTAL_PHYSICAL_MEMORY_SIZE
    * (this.messageStoreConfig.getAccessMessageInMemoryMaxRatio() / 100.0));
getResult.setSuggestPullingFromSlave(diff > memory);
```

其中 maxOffsetPy 为当前最大物理偏移量，maxPhyOffsetPulling 为本次消息拉取最大物理偏移量，他们的差即可表示消息堆积量，TOTAL_PHYSICAL_MEMORY_SIZE 表示当前系统物理内存，accessMessageInMemoryMaxRatio 的默认值为 40，以上逻辑即可算出当前消息堆积量是否大于物理内存的 40 %，如果大于则将 suggestPullingFromSlave 设置为 true。

接下来该参数值会在消息拉取逻辑里面产生作用：

org.apache.rocketmq.broker.processor.PullMessageProcessor#processRequest:

```java
if (getMessageResult.isSuggestPullingFromSlave()) {
  responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
} else {
  responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
}

switch (this.brokerController.getMessageStoreConfig().getBrokerRole()) {
  case ASYNC_MASTER:
  case SYNC_MASTER:
    break;
  case SLAVE:
    if (!this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
      response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
      responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
    }
    break;
}

if (this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
  // consume too slow ,redirect to another machine
  if (getMessageResult.isSuggestPullingFromSlave()) {
    responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
  }
  // consume ok
  else {
    responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
  }
} else {
  responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
}
```

如果发现主服务器的消息堆积超过了物理内存的 40%，则会设置 suggestWhichBrokerId 为从服务器 broker ID。

这里还会有个 slaveReadEnable 值来决定是否可以从从服务器拉取消息：

1. 如果 slaveReadEnable=true，并且堆积量已经超过物理内存 40%时，则建议从从服务器拉取消息，否则还是从主服务器拉取消息；
2. 如果 slaveReadEnable=false，则消息者只能从主服务器中拉取消息。



org.apache.rocketmq.client.impl.consumer.PullAPIWrapper#updatePullFromWhichNode：

```java
public void updatePullFromWhichNode(final MessageQueue mq, final long brokerId) {
    AtomicLong suggest = this.pullFromWhichNodeTable.get(mq);
    if (null == suggest) {
        this.pullFromWhichNodeTable.put(mq, new AtomicLong(brokerId));
    } else {
        suggest.set(brokerId);
    }
}
```

当消费者收到拉取响应回来的数据后，会将下次建议拉取的 brokerID 缓存起来。下次拉取消息就会从 pullFromWhichNodeTable 中取出拉取 brokerId。





