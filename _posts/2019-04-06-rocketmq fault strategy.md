---
layout: post
title: "RocketMQ消息发送的高可用设计"
categories: RocketMQ
tags: message latency 高可用
author: zch
---

* content
{:toc}
从rocketmq topic的创建机制可知，一个topic对应有多个消息队列，那么我们在发送消息时，是如何选择消息队列进行发送的？假如这时有broker宕机了，rocketmq是如何规避故障broker的？看完这篇文章，相信你会从文中找到答案。













rocketmq在发送消息时，由于nameserver检测broker是否还存活是有延迟的，在选择消息队列时难免会遇到已经宕机的broker，又或者因为网络原因发送失败的，因此rocketmq采取了一些高可用设计的方案，主要通过两个手段：**重试与Broker规避**。



## 重试机制


直接定位到client端发送消息的方法：

org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl：

```java
int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
```

```java
for (; times < timesTotal; times++) {
	// ...
}
```

在client端，发送消息的方式有：同步（SYNC）、异步（ASYNC）、单向（ONEWAY）。

那么可以知道，retryTimesWhenSendFailed决定同步方法重试次数，默认重试次数为3次。

重试机制提高了消息发送的成功率。



## 选择队列的方式

我在这里提出一个问题：

**现在有个由两个broker节点组成的集群，有topic1，默认在每个broker上创建4个队列，分别是：master-a（q0,q1,q2,q3）、master-b（q0,q1,q2,q3），上一次发送消息到master-a的q0队列，此时master-a宕机了，如果继续发送topic1消息，rocketmq如果避免再次发送到master-a？**

以上问题引出了rocketmq发送消息时如何选择队列的一些机制，选择队列有两种方式，通过sendLatencyFaultEnable的值来控制，默认值为false，不启动broker故障延迟机制，值为true时启用broker故障延迟机制。

### 默认机制

sendLatencyFaultEnable=false，消息发送选择队列调用以下方法：

org.apache.rocketmq.client.impl.producer.TopicPublishInfo#selectOneMessageQueue：

```java
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
  if (lastBrokerName == null) {
    return selectOneMessageQueue();
  } else {
    int index = this.sendWhichQueue.getAndIncrement();
    for (int i = 0; i < this.messageQueueList.size(); i++) {
      int pos = Math.abs(index++) % this.messageQueueList.size();
      if (pos < 0)
        pos = 0;
      MessageQueue mq = this.messageQueueList.get(pos);
      if (!mq.getBrokerName().equals(lastBrokerName)) {
        return mq;
      }
    }
    return selectOneMessageQueue();
  }
}
```

这里的lastBrokerName指的是上一次执行消息发送时选择失败的broker，在重试机制下，第一次执行消息发送时，lastBrokerName=null，直接选择以下方法：

org.apache.rocketmq.client.impl.producer.TopicPublishInfo#selectOneMessageQueue：

```java
public MessageQueue selectOneMessageQueue() {
  int index = this.sendWhichQueue.getAndIncrement();
  int pos = Math.abs(index) % this.messageQueueList.size();
  if (pos < 0)
    pos = 0;
  return this.messageQueueList.get(pos);
}
```

sendWhichQueue是一个利用ThreadLocal本地线程存储自增值的一个类，自增值第一次使用Random类随机取值，此后如果消息发送出发重试机制，那么每次自增取值。

此方法直接用sendWhichQueue自增获取值，再与消息队列的长度进行取模运算，取模目的是为了循环选择消息队列。

如果此时选择的队列发送消息失败了，此时重试机制在再次选择队列时lastBrokerName不为空，回到最开始的那个方法，还是利用sendWhichQueue自增获取值，但这里多了一个步骤，与消息队列的长度进行取模运算后，如果此时选择的队列所在的broker还是上一次选择失败的broker，则继续选择下一个队列。

我们再细想一下，如果此时有broker宕机了，在默认机制下很可能下一次选择的队列还是在已经宕机的broker，没有办法规避故障的broker，因此消息发送很可能会再次失败，重试发送造成了不必要的性能损失。

所以rocketmq采用Broker故障延迟机制来规避故障的broker。

### Broker故障延迟机制

sendLatencyFaultEnable=true，消息发送选择队列调用以下方法：

org.apache.rocketmq.client.latency.MQFaultStrategy#selectOneMessageQueue：

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
  // Broker故障延迟机制
  if (this.sendLatencyFaultEnable) {
    try {
      // 自增取值
      int index = tpInfo.getSendWhichQueue().getAndIncrement();
      for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
        // 队列位置值取模
        int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
        if (pos < 0)
          pos = 0;
        MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
        // 校验队列是否可用
        if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
          if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
            return mq;
        }
      }

      // 尝试从失败的broker列表中选择一个可用的broker
      final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
      int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
      if (writeQueueNums > 0) {
        final MessageQueue mq = tpInfo.selectOneMessageQueue();
        if (notBestBroker != null) {
          mq.setBrokerName(notBestBroker);
          mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
        }
        return mq;
      } else {
        // 从失败条目中移除已经恢复的broker
        latencyFaultTolerance.remove(notBestBroker);
      }
    } catch (Exception e) {
      log.error("Error occurred when selecting message queue", e);
    }

    return tpInfo.selectOneMessageQueue();
  }

  // 默认机制
  return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

该方法利用sendWhichQueue的自增取值的方式轮询选择队列，与默认机制一致，不同的是多了判断是否可用，调用了latencyFaultTolerance.isAvailable(mq.getBrokerName())判断，其中肯定内涵机关，所以我们需要从延迟机制的几个核心类找突破口。

下面我会从源码的角度详细地分析rocketmq是如何实现在一定时间内规避故障broker的。

从发送消息方法源码看出，在发送完消息，会调用updateFaultItem方法：

org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl：

```java
// 3.执行真正的消息发送
sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
endTimestamp = System.currentTimeMillis();
this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
```

发送消息时捕捉到异常同样会调用updateFaultItem方法，它是延迟机制的核心方法：

```java
endTimestamp = System.currentTimeMillis();
this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
```

其中endTimestamp - beginTimestampPrev等于消息发送需要用到的时间，如果成功发送第三个参数传的是false，发送失败传true，下面继续看updateFaultItem方法的实现源码：

org.apache.rocketmq.client.latency.MQFaultStrategy#updateFaultItem：

```java
public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
  if (this.sendLatencyFaultEnable) {
    long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
    this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
  }
}
```

```java
private long computeNotAvailableDuration(final long currentLatency) {
  for (int i = latencyMax.length - 1; i >= 0; i--) {
    if (currentLatency >= latencyMax[i])
      return this.notAvailableDuration[i];
  }
  return 0;
}
```

其中参数currentLatency为本次消息发送的延迟时间，isolation表示broker是否需要规避，所以消息成功发送表示broker无需规避，消息发送失败时表示broker发生故障了需要规避。

latencyMax和notAvailableDuration是延迟机制算法的核心值，每次发送消息的延迟，它们也决定了失败条目中的startTimestamp的值。

从方法可看出，如果broker需要隔离，消息发送延迟时间默认为30s，再利用这个时间从latencyMax尾部向前找到比currentLatency小的数组下标index，如果没有找到就返回0，我们看看latencyMax和notAvailableDuration这两个数组的默认值：

```java
private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
```

可看出，如果isolation=true，该broker会得到一个10分钟规避时长，如果isolation=false，那么规避时长就得看消息发送的延迟时间是多少了，我们继续往下撸：

org.apache.rocketmq.client.latency.LatencyFaultToleranceImpl#updateFaultItem：

```java
public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
  // 从缓存中获取失败条目
  FaultItem old = this.faultItemTable.get(name);
  if (null == old) {
    // 如果缓存不存在，新建失败条目
    final FaultItem faultItem = new FaultItem(name);
    faultItem.setCurrentLatency(currentLatency);
    // broker开始可用时间=当前时间+规避时长
    faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);

    old = this.faultItemTable.putIfAbsent(name, faultItem);
    if (old != null) {
      // 更新旧的失败条目
      old.setCurrentLatency(currentLatency);
      old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
    }
  } else {
    // 更新旧的失败条目
    old.setCurrentLatency(currentLatency);
    old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
  }
}
```

FaultItem为存储故障broker的类，称为失败条目，每个条目存储了broker的名称、消息发送延迟时长、故障规避开始时间。

该方法主要是对失败条目的一些更新操作，如果失败条目已存在，那么更新失败条目，如果失败条目不存在，那么新建失败条目，其中失败条目的startTimestamp为当前系统时间加上规避时长，startTimestamp是判断broker是否可用的时间值：

org.apache.rocketmq.client.latency.LatencyFaultToleranceImpl.FaultItem#isAvailable：

```java
public boolean isAvailable() {
  return (System.currentTimeMillis() - startTimestamp) >= 0;
}
```

如果当前系统时间大于故障规避开始时间，说明broker可以继续加入轮询的队伍里了。

## 写在最后

经过一波源码的分析，我相信你已经找到了你想要的答案，这个故障延迟机制真的是一个很好的设计，我们看源码不仅仅要看爽，爽了之后我们还要思考一下这些优秀的设计思想在平时写代码的过程中是否可以借鉴一下？

