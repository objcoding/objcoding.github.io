---
layout: post
title: "Kafka/RocketMQ 多线程消费时如何保证消费顺序？"
categories: Kafka RocketMQ
tags: message sequential
author: 张乘辉
---

* content
{:toc}
上两篇文章都在讨论顺序消息的一些知识，看到有个读者的留言如下：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200426170035.png)

这个问题问得非常棒，由于在之前的文章中并没有提及到，因此我在这篇文章中单独讲解，本文将从消费顺序性这个问题出发，深度剖析 Kafka/RocketMQ 多线程消费的线程模型。









## Kafka

kafka 的消费类 KafkaConsumer 是非线程安全的，因此用户无法在多线程中共享一个 KafkaConsumer 实例，且 KafkaConsumer 本身并没有实现多线程消费逻辑，如需多线程消费，还需要用户自行实现，在这里我会讲到 Kafka 两种多线程消费模型。

1、每个线程维护一个 KafkaConsumer

这样相当于一个进程内拥有多个消费者，也可以说消费组内成员是有多个线程内的 KafkaConsumer 组成的。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200426193745.png)

但其实这个消费模型是存在很大问题的，从消费消费模型可看出每个 KafkaConsumer 会负责固定的分区，因此无法提升单个分区的消费能力，如果一个主题分区数量很多，只能通过增加 KafkaConsumer 实例提高消费能力，这样一来线程数量过多，导致项目 Socket 连接开销巨大，项目中一般不用该线程模型去消费。

2、单 KafkaConsumer 实例 + 多 worker 线程

针对第一个线程模型的缺点，我们可采取 KafkaConsumer 实例与消息消费逻辑解耦，把消息消费逻辑放入单独的线程中去处理，线程模型如下：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200426195213.png)

从消费线程模型可看出，当 KafkaConsumer 实例与消息消费逻辑解耦后，我们不需要创建多个 KafkaConsumer 实例就可进行多线程消费，还可根据消费的负载情况动态调整 worker 线程，具有很强的独立扩展性，在公司内部使用的多线程消费模型就是用的单 KafkaConsumer 实例 + 多 worker 线程模型。

但这个消费模型由于消费逻辑是利用多线程进行消费的，因此并不能保证其消息的消费顺序，在这里我们可以引入阻塞队列的模型，一个 woker 线程对应一个阻塞队列，线程不断轮训从阻塞队列中获取消息进行消费，对具有相同 key 的消息进行取模，并放入相同的队列中，实现顺序消费， 消费模型如下：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20200426210045.png)

但是以上两个消费线程模型，存在一个问题：

在消费过程中，如果 Kafka 消费组发生重平衡，此时的分区被分配给其它消费组了，如果还有消息没有消费，虽然 Kakfa 可以实现 ConsumerRebalanceListener 接口，在新一轮重平衡前主动提交消费偏移量，但这貌似解决不了未消费的消息被打乱顺序的可能性？

因此在消费前，还需要主动进行判断此分区是否被分配给其它消费者处理，并且还需要锁定该分区在消费当中不能被分配到其它消费者中（但 kafka 目前时做不到这一点）。

参考 RocetMQ 的做法：

在消费前主动调用 ProcessQueue#isDropped 方法判断队列是否已过期，并且对该队列进行加锁处理（向 broker 端请求该队列加锁）。



## RocketMQ

RocketMQ 不像 Kafka 那么“原生”，RocketMQ 早已为你准备好了你的需求，它本身的消费模型就是单 consumer 实例 + 多 worker 线程模型，有兴趣的小伙伴可以从以下方法观摩 RocketMQ 的消费逻辑：

```java
org.apache.rocketmq.client.impl.consumer.PullMessageService#run
```

RocketMQ 会为每个队列分配一个 PullRequest，并将其放入 pullRequestQueue，PullMessageService 线程会不断轮询从 pullRequestQueue 中取出 PullRequest 去拉取消息，接着将拉取到的消息给到 ConsumeMessageService 处理，ConsumeMessageService 有两个子接口：

```java
// 并发消息消费逻辑实现类
org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService;
// 顺序消息消费逻辑实现类
org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService;
```

其中，ConsumeMessageConcurrentlyService 内部有一个线程池，用于并发消费，同样地，如果需要顺序消费，那么 RocketMQ 提供了 ConsumeMessageOrderlyService 类进行顺序消息消费处理。

经过对 Kafka 消费线程模型的思考之后，从 ConsumeMessageOrderlyService 源码中能够看出 RocketMQ 能够实现局部消费顺序，我认为主要有以下两点：

**1）RocketMQ 会为每个消息队列建一个对象锁，这样只要线程池中有该消息队列在处理，则需等待处理完才能进行下一次消费，保证在当前 Consumer 内，同一队列的消息进行串行消费。**

**2）向 Broker 端请求锁定当前顺序消费的队列，防止在消费过程中被分配给其它消费者处理从而打乱消费顺序。**

 

## 总结

经过这篇文章的分析后，尝试回答文章开头的那个问题：

1）多分区的情况下：

如果想要保证 Kafka 在消费时要保证消费的顺序性，可以使用每个线程维护一个 KafkaConsumer 实例，并且是一条一条地去拉取消息并进行消费（防止重平衡时有可能打乱消费顺序）；对于能容忍消息短暂乱序的业务（话说回来， Kafka 集群也不能保证严格的消息顺序），可以使用单 KafkaConsumer 实例 + 多 worker 线程 + 一条线程对应一个阻塞队列消费线程模型。

1）单分区的情况下：

由于单分区不存在重平衡问题，以上两个线程模型的都可以保证消费的顺序性。



另外如果是 RocketMQ，使用 MessageListenerOrderly 监听消费可保证消息消费顺序。

很多人也有这个疑问：既然 Kafka 和 RocketMQ 都不能保证严格的顺序消息，那么顺序消费还有意义吗？

一般来说普通的的顺序消息能够满足大部分业务场景，如果业务能够容忍集群异常状态下消息短暂不一致的情况，则不需要严格的顺序消息。 

如果你对文章还有什么疑问和补充或者发现文中有错误的地方，欢迎留言，我们一起探讨。