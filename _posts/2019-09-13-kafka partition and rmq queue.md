---
layout: post
title: "Kafka分区副本与RocketMQ队列的区别"
categories: Kafka RocketMQ
tags: 分区 队列 副本 master slave
author: zch
---

* content
{:toc}
最近在学习 Kafka，发现其核心概念与 RocketMQ 还是存在一定的差别，下面我来说下 Kafka 分区 与 RocketMQ

队列之间的区别。









## RocketMQ 队列

RocketMQ 每个主题都会有若干个队列，分布于集群中各个 broker 上，分布规律如下：

 ![](https://gitee.com/objcoding/md-picture/raw/master/img/rocketmq_4.png)

队列会在 broker 中抽象成一个 consumer queue，在集群模式下，每个队列每个消费组只能存在一个消费者进行订阅消费，但是一个消费者可以消费多个队列，这也保证了在集群模式下消息不会被重复消费，如下图所示：

 ![](https://gitee.com/objcoding/md-picture/raw/master/img/rocketmq_12.png)

在RocketMQ开源版本中，在创建主题时，通过集群创建模式，指定主题在集群中的队列数量，比如集群中有 2 个 broker，我们创建主题时选择队列数量为 4，就会在每个 broker 中为该主题创建 4 个 队列，那么该主题在集群中就会有 4 * 2 个队列数量，这里有个不好的地方就是无法精确控制队列数量，但这个问题不大。

RocketMQ 是通过主从模式实现消息的冗余，在生产环境中，也会采取多 Master 多 Slave 模式搭建集群，主从之间的队列数据同步有同步复制和异步复制两种。

因此，RocketMQ 是依靠队列进行消费的，而队列数据通过主从同步实现消息的冗余。



## Kafka分区与副本

Kafka 的分区概念是其核心概念之一，分区机制使得 Kafka 具备了水平扩展的能力，在其分区之上，Kafka 还可以设置分区的副本，大大提高了 Kafka 消息的可靠性。

在 Kafka 中，一个主题在集群中会拥有一个以上分区，每个分区在每个消费集群中只能有一个消费者进行订阅消费，，但是一个消费者可以消费多个队列，与 RocketMQ 队列一样：

 ![](https://gitee.com/objcoding/md-picture/raw/master/img/kafka_1.png)

我们可以通过调整主题的分区数量提高消息的吞吐量，还可以为分区设置副本因子，即该分区在集群中拥有多少个副本（replica），副本分为 leader replica 与 follower replica，它们之间通过 ISR（in-sync replica）与 leader replica 保持数据同步。

在创建主题topic-demo时，可以指定主题在集群中的分区数量，以及副本因子大小：

```bash
--partitions 4 --replication-factor 2
```

以上参数为该主题创建了 4 个分区，副本因子为 2，我现在有个集群，有 3 个 broker：

```bash
nodel brokerid=O 
node2 brokerid=l 
node3 brokerid=2
```

根据 Kafka 的默认分配：

```bash
node1: topic-demo-0、topic-demo-1
node2: topic-demo-1、topic-demo-2、topic-demo-3
node3: topic-demo-0、topic-demo-2、topic-demo-3
```

有没有发现，每个分区都分配了一个副本，而且分区的分布尽量均衡，分区副本尽量不在同一个节点上，如果我们设置副本因子为 3，原理一样。

不同于 RocketMQ 队列，Kafka 的分区可以在集群中精确设置多少个，然后随机均衡地分布在集群上，还可以自由定义副本的多少，而 RocketMQ 的 Master-Slave 模式看起来仅有一份副本，当然为了节省存储空间以及提高性能，一般副本因子设置 2 也就够了。

相对比 RocketMQ 的队列与主从同步机制，Kafka 的分区与副本机制显得更加灵活，而且也更加合理。







