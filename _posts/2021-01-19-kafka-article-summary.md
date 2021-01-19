---
layout: post
title: "花了一年多的时间，写了 30 篇关于 Kafka 的文章"
categories: Kafka
tags: kafka
author: 张乘辉
---

* content
{:toc}
在过去的一年多，由于工作的原因我接触 Kafka 比较多，在工作的过程中，总结了很多关于 Kafka 的知识并将它们沉淀为一篇篇文章，包括 Kafka 核心知识点的讲解，工作中遇到的问题排查分析与性能调优相关，仔细看了下应该有 30 篇了，因此我将这一年多写的 Kafka 文章汇总起来，对 Kafka 进行一次知识梳理。





## 知识点汇总

讲解了 Kafka 副本之间复制日志的一些事，也是我的经典之作，值得品味，同时日志复制也是 Kafka 最核心的概念之一：

[图解：Kafka 水印备份机制](https://mp.weixin.qq.com/s/WSdebVgIpvJ_c4DpFYqO4w)

从缓存申请到缓存归还详细剖析了 Kafka Producer 的缓冲池，同时详细说明了关于缓冲池调优的一些关键性要点：

[深度剖析 Kafka Producer 的缓冲池机制【图解 + 源码分析】](https://mp.weixin.qq.com/s/P6BO5KoMl_NQAI_OcwnXrQ)

从 Kafka 消费组模型概念中讲解了 Kafka 消费组重平衡机制的作用到涉及的一些参数设置注意事项，最后用流程图举例了重平衡的几个场景：

[Kafka重平衡机制](https://mp.weixin.qq.com/s/4DFup_NziFJ1xdc4bZnVcg)

讲述了 Kafka ISR 副本同步机制的要点以及后续升级的设计变动：

[Kafka ISR 副本同步机制](https://mp.weixin.qq.com/s/-uHOUT-AErUnuLjqhvsOow)

关于 Kafka V0、V1、V2 版本的消息格式的一些变动以及这些变动解决了哪些问题，这篇文章都有详细的描述：

[一文看懂 Kafka 消息格式的演进](https://mp.weixin.qq.com/s/XWk0y0XWYmDgIIejtmLUow)

想要快速了解 Kafka 底层存储文件长什么样子？我建议你看下这篇文章：

[Kafka 消息存储与索引设计](https://mp.weixin.qq.com/s/5UZcm9nMEwlSNjrBlrzIVQ)

对于初学者，可能傻傻分不清 RocketMQ 队列和 Kafka 分区副本的区别是什么，这篇文章可以为你解答：

[Kafka分区副本与RocketMQ队列的区别](https://mp.weixin.qq.com/s/bcPJppZUq5lfg09QSB7ooA)

很多用户问起我能不能单独让某些用户只消费指定的分区？而不是由 Kafka 去决定？我回答他 Kafka 是可以做到的，然后我就推荐这位用户看了这篇文章：

[Kafka 独立消费者](https://mp.weixin.qq.com/s/KeM32onJPde1Vv-EwRwiLQ)

现有的消息引擎，想要做到严格的顺序消费，几乎不可能做到，而且代价非常大，以下三篇文章都是关于顺序消费的文章，不一定说得都对，同时也是我在工作过程中学习总结出来的：

[保证严格的消息顺序消费究竟有多难？](https://mp.weixin.qq.com/s/kanPIV58hwe5R3BUmxe8NQ)

[盘点 Kafka 不能保证严格消费顺序的各种情况](https://mp.weixin.qq.com/s/BaRAbk5Zeg9VhCQlhg6ACA)

[Kafka/RocketMQ 多线程消费时如何保证消费顺序？](https://mp.weixin.qq.com/s/lX1xFFEYX6N3eF6lA3R5TQ)

Kafka 分区重分配是一个非常重的操作，一般来说集群扩容都会涉及到分区重分配，那么它的流程究竟是怎样的？我从源码的角度出发，带你揭开它的神秘面纱：

[Kafka 分区重分配源码分析](https://mp.weixin.qq.com/s/6BK28kf2m4ZWKzI2ZjILhw)

有些时候我们不得不对集群中某些废弃的主题进行删除操作，毕竟主题太多，会影响集群性能，那么 Kafka 删除过程是怎么样的？这篇文章或许可以给你答案：

[Kafka 删除主题流程分析](https://mp.weixin.qq.com/s/F7vX5dOspv3yaJKRDiD6zA)

老运维了：

[Kafka 常用运维脚本](https://mp.weixin.qq.com/s/KlXi6brgS6spr4BsYZ2ruA)



## 问题排查汇总

这篇文章为公司双十一前打了一剂强心剂，同时这篇文章在 csdn 上高达 1.4W 的阅读，是什么让这篇文章充满着诱惑力？

[记一次 Kafka 集群线上扩容](https://mp.weixin.qq.com/s/n2dMrs21nUU15Vza0VV1pA)

某次运维老哥因为集群某个节点关闭等待时间太长了，直接 kill -9 暴力解决，导致该节点重启失败，又恰好导致某个分区不可用了，为了找出具体原因我花了周末两天时间去排查，我把排查过程与重启失败原因到最后如何尽量避免消息丢失用了三篇文章去描述，其中我是如何尽量避免消息丢失的具体操作，只有深入 Kafka 相关机制后你才会懂的操作，值得品味与学习：

[记一次 Kafka 重启失败问题排查](https://mp.weixin.qq.com/s/ee7_mhxnj05DxK3EJihyfQ)

[从源码和日志文件结构中分析 Kafka 重启失败事件](https://mp.weixin.qq.com/s/zbwGLygjvO_ncgp7FH9QMA)

[当 Kafka 分区不可用且 leader 副本被损坏时，如何尽量减少数据的丢失？](https://mp.weixin.qq.com/s/b1etPGC97xNjmgdQbycnSg)

当缓冲池满了，当然就会阻塞啦，好像没啥可说的，但你就不想知道它是怎么具体是阻塞的吗？为了验证它会阻塞的事实，我把它的整个过程的写下来了：

[Kafka Producer 异步发送消息居然也会阻塞？](https://mp.weixin.qq.com/s/wbTIW3MkCxaCb8ToFe5wiA)

这篇文章描述了 max.poll.interval.ms 和 max.poll.records 之间的联系：

[一次 kafka 消息堆积问题排查](https://mp.weixin.qq.com/s/VgXukc39tFBXrR0yKg7vdA)

这篇文章描述了 batch.size 和 max.request.size 之间的联系

[Kafka发送消息时提示请求数据过大是怎么回事？](https://mp.weixin.qq.com/s/RhL5E_Dw4QXZgZfb25WqfA)



## 性能调优汇总

从三个维度独创性地设计了一个能够大幅提高 Kafka 顺序消费吞吐量的线程模型，用 6 千字详细描述了整个设计过程：

[Kafka 顺序消费线程模型的实践与优化](https://mp.weixin.qq.com/s/LHlcLTs5r8lQtG8Tl9aPjQ)

Kafka 消息大小的设置还是挺复杂的一件事，以下文章可以为你排忧解难：

[彻底搞懂 Kafka 消息大小相关参数设置的规则](https://mp.weixin.qq.com/s/Yvp3lIwSG3b7LyyBH6FtNA)

[Kafka消息体大小设置的一些细节](https://mp.weixin.qq.com/s/Dv5KDf9AJpAu8t9LtURvCg)

吞吐量上不去，老是被用户质疑集群性能问题？也许可以把这篇文章分享给他：

[记一次 Kafka Producer 性能调优实战](https://mp.weixin.qq.com/s/EKXHWnQIO5SNwdarwfxB2A)

中通消息服务运维平台支撑了中通日均 1000+ 亿的消息流转量，你不想了解它内部消费线程是如何实现的吗？

[Kafka 消费线程模型在中通消息服务运维平台的应用](https://mp.weixin.qq.com/s/O0-wdarb4V--LSuUKbOT2A)



## 面试精选

老面试官了：

[关于 Kafka 的一些面试题目](https://mp.weixin.qq.com/s/AJ45e4TgLDRLJT2ODoQNpw)




