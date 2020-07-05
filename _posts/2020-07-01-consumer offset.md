---
layout: post
title: "记一次关于位移提交的问题回答"
categories: Kafka RocketMQ
tags: offset
author: zch
---

* content
{:toc}
今晚撸得正兴奋时，有个朋友突然问了我一个关于位移提交的问题，他最近刚接触 Kafka，在一篇博客中看到了这么一段话：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200701222319.png)

然后他给我举了不是那么常规的一个问题，如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200701222011.png)

我一看问题就觉得有点奇怪了，我知道这个朋友肯定是从 RocketMQ 过来的，因为在 RocketMQ 的位移提交机制，只能是提交已消费的最小位移：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200701222721.png)

具体我有一篇文章专门详细地分析了 RocketMQ 的位移提交机制：[RocketMQ 位移提交源码分析](https://mp.weixin.qq.com/s/vgfgUT5z2wv8tkFj-5q7lQ)

因此，RocketMQ 是不会发生上面所说的情况。

我觉得产生这种疑惑是因为之前使用 RocketMQ 的时候，由于不用自己处理位移提交，一切交给 RocketMQ 处理了，而恰好 RocketMQ 提交位移的机制只能提交未消费最小偏移量以杜绝消息的丢失，导致了这位朋友切换到 kafka 需要手动处理位移的时候，产生了以上的困惑。

对 Kafka 来说，它提供了手动位移提交的机制，可以暴露出来让用户自行实现位移的提交，也就意味着你可以对分区的位移有控制权，这完全取决于你本身的实现逻辑。

如果是按照例子的描述操作，此时分区最新消费偏移量就是 7 消息的位移，因为 Kafka 它本身并没有重试对列机制，基于这个前提下，如果这条消息消费失败了，要么你客户端捕捉到再进行重试消费，要么就丢弃，消费后面的消息，并提交消费位移，一切都往前看，要不然你会阻塞后面的消费。此时，4 消息就丢失了。

可以这么解决：自己实现一个与 RocketMQ 位移提交机制的 TreeMap 来存储消息，位移作 key，每次消费完移除，提交位移的时候只提交最小位移就好了，比如这个例子，只能提交 3 消息的位移。