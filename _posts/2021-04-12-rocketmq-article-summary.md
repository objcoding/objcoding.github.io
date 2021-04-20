---
layout: post
title: "有关 RocketMQ 的文章都在这里了，请查阅！"
categories: RocketMQ
tags: 
author: 张乘辉
---

* content
{:toc}
过去工作经历接触了比较多消息中间件，也产出了不少关于这方面的技术文章，今天我就来给大家梳理一下本公众号过去关于 RocketMQ 的文章，做一个汇总。









1、很多初学 RocketMQ 的读者向我提问关于 RocketMQ 消费模式的问题， RocketMQ 的消费模式有集群模式和广播模式，这篇文章可以回答这类问题：

[RocketMQ 的消费模式](https://mp.weixin.qq.com/s/gwUuyYdqFcWSqnC2iMqUBA)

2、Kafka 后续的版本有去除 zk 注册中心依赖的的发展趋势，看看 RocketMQ 是怎么做到的：

[RocketMQ源码分析之路由中心](https://mp.weixin.qq.com/s/smTJVoAa8zRDhV2eXQL44A)

3、RocketMQ 的 topic 创建有一些细节需要注意的地方，稍微不注意就会造成性能影响：

[深度解析RocketMQ Topic的创建机制](https://mp.weixin.qq.com/s/muItQvNiNaLWJ3FAbTHitw)

4、RocketMQ 的 topic 创建时，你有没有考虑过队列的分配问题？你是如何考虑主题队列数量的？我是这么思考这些问题的：

[关于RocketMQ Topic的创建机制，我还有一些细节上的思考](https://mp.weixin.qq.com/s/aMmLiMkt82S729BIp8B75Q)

5、向 Broker 发送消息时，难免会碰到 Broker 不可用的时候，可以了解下 RocketMQ 是如何处理这个问题的：

[RocketMQ消息发送的高可用设计](https://mp.weixin.qq.com/s/x5DUxhLexOB7j7vWCi8wSw)

6、如果同一个消费组下，消费者之间的订阅关系不一致，会导致订阅关系不存在的错误发生，我从源码分析得出为什么会产生这个问题的原因：

[RocketMQ为什么要保证订阅关系的一致性？](https://mp.weixin.qq.com/s/8fB-Z5oFPbllp13EcqC9dw)

7、RocketMQ 队列是基于主从模式， Kafka 的分区副本是基于 leader-follower 模式： 

[Kafka分区副本与RocketMQ队列的区别](https://mp.weixin.qq.com/s/bcPJppZUq5lfg09QSB7ooA)

8、Broker 启动过程中的过程中都做了哪些事情？

[RocketMQ Broker启动流程梳理](https://mp.weixin.qq.com/s/Xw9G-wvKZzqyizT5OalTiA)

9、从源码的角度深度剖析了主从同步的实现原理，这部分内容也是 RockeMQ 核心机制之一：

[RocketMQ主从同步源码分析](https://mp.weixin.qq.com/s/y25JgBIKKb6DpTzDMC-Veg)

10、RocketMQ 的主从读写分离机制与 Mysql 有什么区别？这篇文章可以给予你答案：

[RocketMQ主从读写分离机制](https://mp.weixin.qq.com/s/duweizStvKkCbrY03g2zkg)

11、消费进度是如何在主从节点直接同步的？集群模式和广播模式都有所不同：

[RocketMQ主从如何同步消息消费进度？](https://mp.weixin.qq.com/s/3BiIvcZ6RPRKNiM-4eCs9Q)

12、消息消费前，需要从 Broker 节点拉取消息，那么拉取消息会有 Push 模式和 Pull 模式，RocketMQ 的消息拉取机制是如何实现的？这是 RocketMQ 很重要的知识点之一：

[关于RocketMQ消息拉取与重平衡的一些问题探讨](https://mp.weixin.qq.com/s/ZsgrNsYPhKbssIaOVeDMfw)

13、异常的意思是从节点不可用，那么为什么会导致从节点不可用？这里从节点不一定挂了，而是因为从节点的消费偏移量落后主节点太多导致的，以下这篇文章详细地解读了产生这个异常报错的原因：

[RocketMQ 同步复制 SLAVE_NOT_AVAILABLE 异常源码分析](https://mp.weixin.qq.com/s/WPL5ch8mtjNGLrFr4L1LqA)

14、新手入门必备：

[搭建 RocketMQ 集群](https://mp.weixin.qq.com/s/qyhnbQ-OefAPk9dF9iw-Lg)

15、并发消费的时候，一次从 一个队列拉 32 条消息，这 32 条消息会提交到线程池中处理，如果偏移量  m5 比 m4 先执行完成，消息消费后，提交的消费进度是哪个？是提交消息 m5 的偏移量？

[RocketMQ 位移提交源码分析](https://mp.weixin.qq.com/s/vgfgUT5z2wv8tkFj-5q7lQ)

16、无论是 RocketMQ 还是 Kafka，都不能严格地保证消息消费的顺序性，在允许一定的范围内，RocketMQ 自身已有顺序消费的逻辑实现：

[Kafka/RocketMQ 多线程消费时如何保证消费顺序？](https://mp.weixin.qq.com/s/lX1xFFEYX6N3eF6lA3R5TQ)

17、Kafka 和 RocketMQ 的位移提交对比，由于 Kafka 不支持消息重试，因此在消费时需要客户端保证消息消费的过程是成功的，这也是我在过往中遇比较多读者问的问题：

[记一次关于位移提交的问题回答](https://mp.weixin.qq.com/s/sPyiVkbWSVNViHGQqSy2Pg)

