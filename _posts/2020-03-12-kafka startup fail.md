---
layout: post
title: "记一次 Kafka 重启失败问题排查"
categories: Kafka
tags: log
author: zch
---

* content
{:toc}


## 背景

在 2 月10 号下午大概 1 点半左右，收到用户方反馈，发现日志 kafka 集群 A 主题 的 34 分区选举不了 leader，导致某些消息发送到该分区时，会报如下 no leader 的错误信息：

```
In the middle of a leadership election, there is currently no leader for this partition and hence it is unavailable for writes.
```

接下来运维在 kafka-manager 查不到 broker0 节点了处于假死状态，但是进程依然还在，重启了好久没见反应，然后通过 kill -9 命令杀死节点进程后，接着重启失败了，导致了如下两个问题：

1. 由于 A 主题 34 分区的 leader 副本在 broker0，另外一个副本由于速度跟不上 leader，已被踢出 ISR，0.11 版本的 kafka 的 unclean.leader.election.enable 参数默认为 false，表示分区不可在 ISR 以外的副本选举 leader，导致了 A 主题发送消息持续报 34 分区 leader 不存在的错误，如果发送重试只有一次，这很可能会造成消息丢失的风险；
2. 33 节点还存在很多没消费的消息，会导致这部分日志查询不了。



## Kafka 日志分析

查看了 KafkaServer.log 日志，发现 Kafka 重启过程中，产生了大量如下日志：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200312212507.png)

发现大量主题索引文件损坏并且重建索引文件的警告信息，定位到源码处：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200311204129.png)

Kafka 在启动的时候，会检查kafka是否为 cleanshutdown，判断依据为 ${log.dirs} 目录中是否存在 .kafka_cleanshutDown 的文件，如果非正常退出就没有这个文件，接着就需要 recover log 处理，在处理中会调用 sanityCheck() 方法用于检验每个 log sement 的 index 文件，确保索引文件的完整性：

- entries：由于 kafka 的索引文件是一个稀疏索引，并不会将每条消息的位置都保存到 .index 文件中，因此引入了 entry 模式，即每一批消息只记录一个位置，因此索引文件的 entries = mmap.position / entrySize；

- lastOffset：最后一块 entry 的位移，即 lastOffset = lastEntry.offset；

- baseOffset：指的是索引文件的基偏移量，即索引文件名称的那个数字。

判断索引文件是否损坏的依据是：

```scala
_entries == 0 || _lastOffset > baseOffset = false // 损坏
_entries == 0 || _lastOffset > baseOffset = true // 正常
```

这个判断逻辑我的理解是：

entries 等于零时，意味着索引没有内容，此时 lastOffset 与基偏移量相等，二者相减等零才对，但相减索引的长度不是零，所以判断索引就损坏了。

那为什么会出现这种情况呢？

前面我也说过了， kafka 的索引文件是一个稀疏索引，也正因为这机制，在节点退出时需要对索引文件进行 compacted 处理，如果是非正常退出，很可能就会造成索引文件的位置信息不对应了，因此在节点非正常退出时，都会出现这个日志出现。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200311195627.png)

有意思的来了，导致开机不了并不是这个问题导致的，因为这个问题已经在后续版本修复了，从日志可看出，它会将损坏的日志文件删除并重建，我们接下来继续看导致重启不了的错误信息：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200312212611.png)

问题就出在这里，在**删除并重建索引过程中，就可能出现如上问题**，在 issues.apache.org 网站上有很多关于这个 bug 的描述，我这里贴两个出来：

https://issues.apache.org/jira/browse/KAFKA-4972

https://issues.apache.org/jira/browse/KAFKA-3955

这些 bug 很隐晦，而且非常难复现，既然后续版本不存在该问题，当务之急还是升级 Kafka 版本，后续等我熟悉 scala 后，再继续研究下源码，细节一定是会在源码中呈现。



## 解决思路分析

针对背景两个问题，矛盾点都是因为 33 节点重启失败导致的，那么我们要么把 33 节点启动成功，要么把 33 节点从集群中排除掉。

### 方案一：删除磁盘日志文件

该方案就是将 broker0 磁盘文件和索引文件删除，然后重新启动，但这个方案有个问题需要解决：

由于 logKafka 集群节点有一个参数：unclean.leader.election.enable 参数的值设置为 false，这个参数就是为了控制主题分区 leader 选举只能从 ISR 集合中选举，而这也就导致了 logkafka 的 A 主题 的 34 分区选举不了 leader 的根本原因，那么也就是意味着，即使你重新启动 broker0，A 主题 的 34 分区依然举不了 leader，因为没有副本可以同步了，永远找不到 leader 了。

可以在 broker0 启动成功后，手动将 34 分区的 preferred leader 更换 broker 位置。

### 方案二：移除 broker0

kafka 移除节点并不是直接不启动节点就万事大吉了，因为 zk 还是会有该节点的信息和策略，只不过显示了该节点宕机了而已，如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200312212733.png)

在主题的分区策略中，依然还存在 broker0。

因此，移除 broker0 就是需要将该节点上的主题分区日志进行迁移，迁移的方法就是进行分区重分配，在分配策略中不包含 broker0 即可，但是分区重分配是一个重量级操作，过程会很慢，这个可以针对 A 主题优先进行分区重分配。

不得不说，除了过程慢了点，Kafka 分区重分配简直就是一把利器，可以进行扩容，同时还可以进行缩容处理，还能变相移除节点。



## 后续集群的优化

1. 制定一个升级方案，将集群升级到 2.2 版本；
2. 每个节点的服务器将 systemd 的默认超时值为 600 秒，因为我发现运维在故障当天关闭 33 节点时长时间没反应，才会使用 kill -9 命令强制关闭。但据我了解关闭一个 Kafka 服务器时，Kafka 需要做很多相关工作，这个过程可能会存在相当一段时间，而 systemd 的默认超时值为 90 秒即可让进程停止，那相当于非正常退出了。
3. 将 broker 参数 unclean.leader.election.enable 设置为 true；（确保分区可从非 ISR 中选举 leader）
4. 将 broker 参数 default.replication.factor 设置为 3；（提高高可用，但会增大集群的存储压力，可后续讨论）
5. 发送端发送 acks=1（确保发送时有一个副本是同步成功的，但这个是否有必要，因为可能会造成性能损失）



