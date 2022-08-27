---
layout: post
title: "Kafka 删除主题流程分析"
categories: Kafka
tags: 主题 topic
author: 张乘辉
---

* content
{:toc}
之前有个 Kafka 集群的每个节点的挂载磁盘多达 20+ 个，平均每个磁盘约1T，每个节点的分区日志被平均分配到这些磁盘中，但由于每个分区的数据不一致，而集群节点 log.retention.bytes 这个参数的默认值是 -1，也就是没有任何限制，因此 Kafka 的日志删除日志依赖 log.retention.hours 参数来删除，因此会出现日志未过期，磁盘写满的情况。

针对该集群双十一会遇到某些挂载磁盘被写满的情况，需要手动对主题进行删除以清空磁盘的操作，现在分析删除主题对集群以及客户端会有什么影响，以及 Kafka 都做了哪些动作。









## 图解删除过程

1. 删除主题

删除主题有多种方法，可通过 kafka-topic.sh 脚本并执行 --delete 命令，或者用暴力方式直接在 zk 删除对应主题节点，其实删除主题无非就是令 zk 节点删除，以触发 controller 对应监听器，然后再通过监听器通知到所有 broker，具体流程如下：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20191111073445.png)

删除主题执行后，controller 监听到 zk 主题节点被删除，通知到所有 broker 删除主题对应的副本，这里会分成两个步骤，第一个步骤先将下线主题对应的副本，最后才执行真正的删除操作，注意，这里也并为真正的将主题从磁盘中删除，此时仅仅只会将要删除的副本所在的目录重命名，以免之后创建主题时目录有冲突，每个broker 都会有一个定时线程，定时清除已重命名为删除状态的日志文件，具体如下：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20191111074026.png)



2. 自动创建主题

自动创建主题的前提是 broker 配置参数 auto.create.topic.enble=true，删除主题后，当 Producer 发送时会对发送进行重试，期间会发送 MetadataRquest 命令到 broker 请求获取最新的元数据，在获取元数据的同时，会判断是否需要自动创建主题，如果需要，则调用 zk 客户端创建主题节点，controller 监听到有新主题创建，就会触发 controller 相关状态机工作创建主题。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20191111073545.png)



刚刚也说过，kafka 重命名要删除的主题后，并不会立马就会删除，而是等待异步线程去删除，如下图所示，重命名后与重新创建的分区不冲突，可以证明删除是异步执行的了，且不影响生产发送，但是被重命名后的日志就不能消费了，即丢失了。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20191111074956.png)

如下图可看出，在一分钟后，重命名后的副本被删除。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20191112081724.png)



## 相关日志分析

### 1、controller.log

触发删除主题监听器：

```
[2019-11-07 19:24:11,121] DEBUG [Controller id=0] Delete topics listener fired for topics test-topic to be deleted (kafka.controller.KafkaController)
```

开始删除主题操作：

```
[2019-11-07 19:24:11,121] INFO [Topic Deletion Manager 0] Handling deletion for topics test-topic (kafka.controller.TopicDeletionManager)
```

开始停止主题，但此时并未删除：

```
[2019-11-07 19:24:11,143] DEBUG The stop replica request (delete = false) sent to broker 2 is StopReplicaRequestInfo([Topic=test-topic,Partition=1,Replica=2],false),StopReplicaRequestInfo([Topic=test-topic,Partition=0,Replica=2],false),StopReplicaRequestInfo([Topic=test-topic,Partition=2,Replica=2],false) (kafka.controller.ControllerBrokerRequestBatch)
```

开始执行真正的删除动作：

```
[2019-11-07 19:24:11,145] DEBUG [Topic Deletion Manager 0] Deletion started for replicas

[2019-11-07 19:24:11,147] DEBUG The stop replica request (delete = true) sent to broker 2 is StopReplicaRequestInfo([Topic=test-topic,Partition=1,Replica=2],true),StopReplicaRequestInfo([Topic=test-topic,Partition=0,Replica=2],true),StopReplicaRequestInfo([Topic=test-topic,Partition=2,Replica=2],true) (kafka.controller.ControllerBrokerRequestBatch)
```

收到 broker 删除的回调:

```
[2019-11-07 19:24:11,170] DEBUG [Controller id=0] Delete topic callback invoked on StopReplica response received from broker 2: request error = NONE, partition errors = Map(test-topic-2 -> NONE, test-topic-0 -> NONE, test-topic-1 -> NONE) (kafka.controller.KafkaController)

[2019-11-07 19:24:11,170] DEBUG [Topic Deletion Manager 0] Deletion successfully completed for replicas
```

已经成功全部删除:

```
[2019-11-07 19:24:11,202] INFO [Topic Deletion Manager 0] Deletion of topic test-topic successfully completed (kafka.controller.TopicDeletionManager)
```

如果此时有新的消息写入，会自动创建主题:

```
[2019-11-07 19:24:11,203] INFO [Controller id=0] New topics: [Set()], deleted topics: [Set()], new partition replica assignment [Map()] (kafka.controller.KafkaController)

[2019-11-07 19:24:11,267] INFO [Controller id=0] New topics: [Set(test-topic)], deleted topics: [Set()], new partition replica assignment [Map(test-topic-2 -> Vector(1, 2, 0), test-topic-1 -> Vector(0, 1, 2), test-topic-0 -> Vector(2, 0, 1))] (kafka.controller.KafkaController)

[2019-11-07 19:24:11,267] INFO [Controller id=0] New partition creation callback for test-topic-2,test-topic-1,test-topic-0 (kafka.controller.KafkaController)
```

### 2、server.log

broker 收到删除主题通通知（此时并没有删除）:

```
[2019-11-07 19:24:11,144] INFO [ReplicaFetcherManager on broker 2] Removed fetcher for partitions Set(test-topic-2, test-topic-0, test-topic-1) (kafka.server.ReplicaFetcherManager)
```

停止分区 fetch 线程:

```
[2019-11-07 19:24:11,145] INFO [ReplicaFetcher replicaId=2, leaderId=1, fetcherId=0] Shutting down (kafka.server.ReplicaFetcherThread)

[2019-11-07 19:24:11,146] INFO [ReplicaFetcher replicaId=2, leaderId=1, fetcherId=0] Error sending fetch request (sessionId=293639440, epoch=1824) to node 1: java.io.IOException: Client was shutdown before response was read. (org.apache.kafka.clients.FetchSessionHandler)

[2019-11-07 19:24:11,146] INFO [ReplicaFetcher replicaId=2, leaderId=1, fetcherId=0] Stopped (kafka.server.ReplicaFetcherThread)

[2019-11-07 19:24:11,147] INFO [ReplicaFetcher replicaId=2, leaderId=1, fetcherId=0] Shutdown completed (kafka.server.ReplicaFetcherThread)
```

接收到真正删除主题指令后，会重命名分区日志目录，此时还未删除，会等待异步线程执行:

```
[2019-11-07 19:24:11,157] INFO Log for partition test-topic-2 is renamed to /tmp/kafka-logs/kafka_3/test-topic-2.93ed68ff29d64a01a3f15937859124f7-delete and is scheduled for deletion (kafka.log.LogManager)
```

如果此时有新的消息写入，会自动创建主题:

```
[2019-11-08 15:39:39,343] INFO Creating topic test-topic with configuration {} and initial partition assignment Map(2 -> ArrayBuffer(1, 0, 2), 1 -> ArrayBuffer(0, 2, 1), 0 -> ArrayBuffer(2, 1, 0)) (kafka.zk.AdminZkClient)

[2019-11-08 15:39:39,369] INFO [KafkaApi-1] Auto creation of topic test-topic with 3 partitions and replication factor 3 is successful (kafka.server.KafkaApis)

[2019-11-07 19:24:11,286] INFO Created log for partition test-topic-0 in /tmp/kafka-logs/kafka_3 with properties {...}
```

异步线程删除重命名后的主题:

```
[2019-11-07 19:25:11,161] INFO Deleted log /tmp/kafka-logs/kafka_3/test-topic-2.93ed68ff29d64a01a3f15937859124f7-delete/00000000000000000000.log. (kafka.log.LogSegment)

[2019-11-07 19:25:11,163] INFO Deleted offset index /tmp/kafka-logs/kafka_3/test-topic-2.93ed68ff29d64a01a3f15937859124f7-delete/00000000000000000000.index. (kafka.log.LogSegment)

[2019-11-07 19:25:11,164] INFO Deleted time index /tmp/kafka-logs/kafka_3/test-topic-2.93ed68ff29d64a01a3f15937859124f7-delete/00000000000000000000.timeindex. (kafka.log.LogSegment)

[2019-11-07 19:25:11,165] INFO Deleted log for partition test-topic-2 in /tmp/kafka-logs/kafka_3/test-topic-2.93ed68ff29d64a01a3f15937859124f7-delete. (kafka.log.LogManager)
```



