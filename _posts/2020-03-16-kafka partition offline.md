---
layout: post
title: "当 Kafka 分区不可用且 leader 副本被损坏时，如何尽量减少数据的丢失？"
categories: Kafka
tags: partition 分区 数据 leader副本
author: 张乘辉
---

* content
{:toc}
经过上次 Kafka 日志集群某节点重启失败导致某个主题分区不可用的事故之后，这篇文章专门对分区不可用进行故障重现，并给出我的一些骚操作来尽量减少数据的丢失。









## 故障重现

下面我用一个例子重现现分区不可用且 leader 副本被损坏的例子：

1. 使用 unclean.leader.election.enable = false 参数启动 broker0；
2. 使用 unclean.leader.election.enable = false 参数启动 broker1；
3. 创建 topic-1，partition=1，replica-factor=2；
4. 将消息写入 topic-1；
5. 此时，两个 broker 上的副本都处于 ISR 中，broker0 的副本为 leader 副本；
6. 停止 broker1，此时 topic-1 的 leader 依然时 broker0 的副本，而 broker1 的副本从 ISR 中剔除；
7. **停止 broker0，并且删除 broker0 上的日志数据**；
8. 重启 broker1，topic-1 尝试连接 leader 副本，但此时 broker0 已经停止运行，此时分区处于不可用状态，无法写入消息；
9. 恢复 broker0，broker0 上的副本恢复 leader 职位，**此时 broker1 尝试加入 ISR，但此时由于 leader 的数据被清除，即偏移量为 0，此时 broker1 的副本需要截断日志，保持偏移量不大于 leader 副本，此时分区的数据全部丢失**。



## 我的建议

在遇到分区不可用时，是否可以提供一个选项，让用户可以手动设置分区内任意一个副本作为 leader？

因为集群一旦设置了 unclean.leader.election.enable = false，就无法选举 ISR 以外的副本作为 leader，在极端情况下仅剩 leader 副本还在 ISR 中，此时 leader 所在的 broker 宕机了，那如果此时 broker 数据发生损坏这么办？在这种情况下，能不能让用户自己选择 leader 副本呢？尽管这么做也是会有数据丢失，但相比整个分区的数据都丢失而言，情况还是会好很多的。



## 我的骚操作

首先你得有一个不可用的分区（并且该分区 leader 副本数据已损失），如果是测试，可以以上故障重现 1-8 步骤实现一个不可用的分区（需要增加一个 broker）：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315200806.png)

此时 leader 副本在 broker0，但已经挂了，且分区不可用，此时 broker2 的副本由于掉出 ISR ，不可选为 leader，且 leader 副本已损坏清除，如果此时重启 broker0，follower 副本会进行日志截断，将会丢失该分区所有数据。

经过一系列的测试与实验，我总结出了以下骚操作，可以强行把  broker2 的副本选为 leader，尽量减少数据丢失：

1、使用 kafka-reassign-partitions.sh 脚本对该主题进行分区重分配，当然你也可以使用 kafka-manager 控制台对该主题进行分区重分配，重分配之后如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315201915.png)

此时 preferred leader 已经改成 broker2 所在的副本了，但此时的 leader 依然还是 broker0 的副本。**需要注意的是，分区重分配之后的 preferred leader 一定要之前那个踢出 ISR 的副本，而不是分区重分配新生成的副本。因为新生成的副本偏移量为 0，如果自动重分配不满足，那么需要编写 json 文件，手动更改分配策略。**

2、进入 zk，查看分区状态并修改它的内容：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315202132.png)

修改 node 内容，强行将 leader 改成 2（与重分配之后的  preferred leader 一样），并且将 leader_epoch 加 1 处理，同时 ISR 列表改成 leader，改完如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315202637.png)

此时，kafka-manager 控制台会显示成这样：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315202812.png)

但此时依然不生效，记住这时需要重启 broker 0。

3、重启 broker0，发现分区的 lastOffset 已经变成了  broker2 的副本的 lastOffset：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315203052.png)

成功挽回了 46502 条消息数据，尽管依然丢失了 76053 - 46502 = 29551 条消息数据，但相比全部丢失相对好吧！

以上方法的原理其实很简单，就是强行把 Kafka 认定的 leader 副本改成自己想要设置的副本，然后 lastOffset 就会以我们手动设置的副本 lastOffset 为基准了。
