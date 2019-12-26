---
layout: post
title: "关于RocketMQ消息消费与重平衡的一些问题"
categories: RocketMQ
tags: consumer push pull rebalance
author: zch
---

* content
{:toc}
其实最好的学习方式就是互相交流，最近也有跟网友讨论了一些关于 RocketMQ 消息拉取与重平衡的问题，我姑且在这里写下我的一些总结。







## 关于 push 模式下的消息循环拉取问题

之前发表了一篇关于重平衡的文章：「[Kafka 重平衡机制](https://mp.weixin.qq.com/s/4DFup_NziFJ1xdc4bZnVcg)」，里面有说到 RocketMQ 重平衡机制是每隔 20s 从任意一个 Broker 节点获取消费组的消费 ID 以及订阅信息，再根据这些订阅信息进行分配，然后将分配到的信息封装成 pullRequest 对象 pull 到 pullRequestQueue 队列中，拉取线程唤醒后执行拉取任务，流程图如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/rocketmq_16.png)

但是其中有一些是没有详细说的，比如每次拉消息都要等 20s 吗？真的有个网友问了我如下问题：

![](https://gitee.com/objcoding/md-picture/raw/master/img/rocketmq_17.png)

很显然他的项目是用了 push 模式进行消息拉取，要回答这个问题，就要从 RockeMQ 的消息拉取说起：

RocketMQ 的 push 模式的实现是基于 pull 模式，只不过在 pull 模式上套了一层，所以RocketMQ push 模式并不是真正意义上的 ”推模式“，因此，在 push 模式下，消费者拉取完消息后，立马就有开始下一个拉取任务，并不会真的等 20s 重平衡后才拉取，至于 push 模式是怎么实现的，那就从源码去找答案。

之前有写过一篇文章：「[RocketMQ为什么要保证订阅关系的一致性？](https://mp.weixin.qq.com/s/8fB-Z5oFPbllp13EcqC9dw)」，里面有说过 消息拉取是从 PullRequestQueue 阻塞队列中取出 PullRequest 拉取任务进行消息拉取的，但 PullRequest 是怎么放进 PullRequestQueue 阻塞队列中的呢？

RocketMQ 一共提供了以下方法：

org.apache.rocketmq.client.impl.consumer.PullMessageService#executePullRequestImmediately：

```java
public void executePullRequestImmediately(final PullRequest pullRequest) {
  try {
    this.pullRequestQueue.put(pullRequest);
  } catch (InterruptedException e) {
    log.error("executePullRequestImmediately pullRequestQueue.put", e);
  }
}
```

从调用链发现，除了重平衡会调用该方法之外，在 push 模式下，PullCallback 回调对象中的 onSuccess 方法在消息消费时，也调用了该方法：

org.apache.rocketmq.client.consumer.PullCallback#onSuccess：

case FOUND:

```java
// 如果本次拉取消息为空，则继续将pullRequest放入阻塞队列中
if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
  DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
} else {
  // 将消息放入消费者消费线程去执行
  DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(//
    pullResult.getMsgFoundList(), //
    processQueue, //
    pullRequest.getMessageQueue(), //
    dispathToConsume);
  // 将pullRequest放入阻塞队列中
  DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);  
}

```

当从 broker 拉取到消息后，如果消息被过滤掉，则继续将pullRequest放入阻塞队列中继续循环执行消息拉取任务，否则将消息放入消费者消费线程去执行，在pullRequest放入阻塞队列中。

case NO_NEW_MESSAGE： 

case NO_MATCHED_MSG：

```java
pullRequest.setNextOffset(pullResult.getNextBeginOffset());
DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
```

如果从 broker 端没有可拉取的新消息或者没有匹配到消息，则将pullRequest放入阻塞队列中继续循环执行消息拉取任务。

从以上消息消费逻辑可以看出，当消息处理完后，立即将 pullRequest 重新放入阻塞队列中，因此这就很好解释为什么 push 模式可以持续拉取消息了：

在 push 模式下消息消费完后，还会调用该方法重新将 PullRequest 对象放进 PullRequestQueue 阻塞队列中，不断地从 broker 中拉取消息，实现 push 效果。



## 重平衡后队列被其它消费者分配后如何处理？

继续再想一个问题，如果重平衡后，发现某个队列被新的消费者分配了，怎么办，总不能继续从该队列中拉取消息吧？

RocketMQ 重平衡后会检查 pullRequest 是否还在新分配的列表中，如果不在，则丢弃，调用 isDrop() 可查出该pullRequest是否已丢弃：

org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage：

```java
final ProcessQueue processQueue = pullRequest.getProcessQueue();
if (processQueue.isDropped()) {
  log.info("the pull request[{}] is dropped.", pullRequest.toString());
  return;
}
```

在消息拉取之前，首先判断该队列是否被丢弃，如果已丢弃，则直接放弃本次拉取任务。

那什么时候队列被丢弃呢？

org.apache.rocketmq.client.impl.consumer.RebalanceImpl#updateProcessQueueTableInRebalance：

```java
Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
while (it.hasNext()) {
  Entry<MessageQueue, ProcessQueue> next = it.next();
  MessageQueue mq = next.getKey();
  ProcessQueue pq = next.getValue();

  if (mq.getTopic().equals(topic)) {
    // 判断当前缓存 MessageQueue 是否包含在最新的 mqSet 中，如果不存在则将队列丢弃
    if (!mqSet.contains(mq)) {
      pq.setDropped(true);
      if (this.removeUnnecessaryMessageQueue(mq, pq)) {
        it.remove();
        changed = true;
        log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
      }
    } else if (pq.isPullExpired()) {
      // 如果队列拉取过期则丢弃
      switch (this.consumeType()) {
        case CONSUME_ACTIVELY:
          break;
        case CONSUME_PASSIVELY:
          pq.setDropped(true);
          if (this.removeUnnecessaryMessageQueue(mq, pq)) {
            it.remove();
            changed = true;
            log.error("[BUG]doRebalance, {}, remove unnecessary mq, {}, because pull is pause, so try to fixed it",
                      consumerGroup, mq);
          }
          break;
        default:
          break;
      }
    }
  }
}
```
updateProcessQueueTableInRebalance 方法在重平衡时执行，用于更新 processQueueTable，它是当前消费者的队列缓存列表，以上方法逻辑判断当前缓存 MessageQueue 是否包含在最新的 mqSet 中，如果不包含其中，则说明经过这次重平衡后，该队列被分配给其它消费者了，或者拉取时间间隔太大过期了，则调用 setDropped(true) 方法将队列置为丢弃状态。

可能你会问，processQueueTable 跟 pullRequest 里面 processQueue 有什么关联，往下看：

org.apache.rocketmq.client.impl.consumer.RebalanceImpl#updateProcessQueueTableInRebalance：

```java
// 新建 ProcessQueue 
ProcessQueue pq = new ProcessQueue();
long nextOffset = this.computePullFromWhere(mq);
if (nextOffset >= 0) {
  // 将ProcessQueue放入processQueueTable中
  ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
  if (pre != null) {
    log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
  } else {
    log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
    PullRequest pullRequest = new PullRequest();
    pullRequest.setConsumerGroup(consumerGroup);
    pullRequest.setNextOffset(nextOffset);
    pullRequest.setMessageQueue(mq);
    // 将ProcessQueue放入pullRequest拉取任务对象中
    pullRequest.setProcessQueue(pq);
    pullRequestList.add(pullRequest);
    changed = true;
  }
}
```

可以看出，重平衡时会创建 ProcessQueue 对象，将其放入 processQueueTable 缓存队列表中，再将其放入 pullRequest 拉取任务对象中，也就是 processQueueTable 中的 ProcessQueue 与 pullRequest 的中 ProcessQueue 是同一个对象。



## 重平衡后会导致消息重复消费吗？

之前在群里有个网友提了这个问题：

![](https://gitee.com/objcoding/md-picture/raw/master/img/rocketmq_18.png)

我当时回答他 RocketMQ 正常也是没有重复消费，但后来发现其实 RocketMQ 在某些情况下，也是会出现消息重复消费的现象。

前面讲到，RocketMQ 消息消费时，会将消息放进消费线程中去执行，代码如下：

org.apache.rocketmq.client.consumer.PullCallback#onSuccess：

```java
DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(//
  pullResult.getMsgFoundList(), //
  processQueue, //
  pullRequest.getMessageQueue(), //
  dispathToConsume);
```

ConsumeMessageService 类实现消息消费的逻辑，它有两个实现类：

```java
// 并发消息消费逻辑实现类
org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService;
// 顺序消息消费逻辑实现类
org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService;
```

先看并发消息消费相关处理逻辑：

ConsumeMessageConcurrentlyService：

org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService.ConsumeRequest#run：

```java
if (this.processQueue.isDropped()) {
  log.info("the message queue not be able to consume, because it's dropped. group={} {}", ConsumeMessageConcurrentlyService.this.consumerGroup, this.messageQueue);
  return;
}

// 消息消费逻辑
// ...

// 如果队列被设置为丢弃状态，则不提交消息消费进度
if (!processQueue.isDropped()) {
    ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
} else {
    log.warn("processQueue is dropped without process consume result. messageQueue={}, msgs={}", messageQueue, msgs);
}
```

ConsumeRequest 是一个继承了 Runnable 的类，它是消息消费核心逻辑的实现类，submitConsumeRequest 方法将 ConsumeRequest 放入 消费线程池中执行消息消费，从它的 run 方法中可看出，如果在执行消息消费逻辑中有节点加入，重平衡后该队列被分配给其它节点进行消费了，此时的队列被丢弃，则不提交消息消费进度，因为之前已经消费了，此时就会造成消息重复消费的情况。

再来看看顺序消费相关处理逻辑：

ConsumeMessageOrderlyService：

org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService.ConsumeRequest#run：

```java
public void run() {
  // 判断队列是否被丢弃
  if (this.processQueue.isDropped()) {
    log.warn("run, the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
    return;
  }

  final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
  synchronized (objLock) {
    // 如果不是广播模式，且队列已加锁且锁没有过期
    if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
        || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {
      final long beginTime = System.currentTimeMillis();
      for (boolean continueConsume = true; continueConsume; ) {
        // 再次判断队列是否被丢弃
        if (this.processQueue.isDropped()) {
          log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
          break;
        }
        
        // 消息消费处理逻辑
        // ...
        
          continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
        } else {
          continueConsume = false;
        }
      }
    } else {
      if (this.processQueue.isDropped()) {
        log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
        return;
      }
      ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 100);
    }
  }
}
```

RocketMQ 顺序消息消费会将队列锁定，当队列获取锁之后才能进行消费，所以，即使消息在消费过程中有节点加入，重平衡后该队列被分配给其它节点进行消费了，此时的队列被丢弃，依然不会造成重复消费。

