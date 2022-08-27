---
layout: post
title: "Kafka消息体大小设置的一些细节"
categories: Kafka
tags: message 消息
author: 张乘辉
---

* content
{:toc}
还记得前几天有个小伙伴跟我反馈发送消息时提示请求数据过大的异常吗？经过调整 max.request.size 的大小之后，又报了了如下异常：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/kafka_3.png)







查看相关资料后，发现 Broker 端对 Producer 发送过来的消息也有一定的大小限制，这个参数叫 message.max.bytes，这个参数决定了 Broker 能够接收到的最大消息的大小，它的默认值为 977 KB，而 max.request.size 的值已经设置成 2M 大小了，很显然已经比 message.max.bytes 大了很多，因此消息大于 997KB 时，就会抛出如上异常。 

值得一提的是，主题配置也有一个参数，叫 max.message.bytes，它只针对某个主题生效，可动态配置，可覆盖全局的 message.max.bytes，好处就是可以针对不同主题去设置 Broker 接收消息的大小，而且不用重启 Broker。

这还没完，消费端拉取消息数据的大小也需要更改，这个参数叫 fetch.max.bytes，这个参数决定消费者单次从 Broker 获取消息的最大字节数，那么问题来了，如果该参数值比 max.request.size 小，那么会导致消费者很可能消费不了比 fetch.max.bytes 大的消息。

所以综合起来，需要这么设置：
```
producer端：
max.request.size=5242880（5M）
broker：
message.max.bytes=6291456（6M）
consumer：
fetch.max.bytes=7340032（7M）

max.request.size < message.max.bytes < fetch.max.bytes
```



另外补充一点，还记得之前说过的 batch.size 参数的作用吗，从源码可看出，Producer 每次发送的消息封装成 ProducerRecord，然后利用消息累加器 RecordAccumulator 添加到 ProducerBatch 中，由于每次创建 ProducerBatch 都需要分配一个 batch.size 大小的内存空间，频繁创建和关闭会导致性能极大开销，所以 RecordAccumulator 内部有个 BufferPool，它实现了缓存的复用，只不过只针对 batch.size 大小的 BufferByte 进行复用，如果大于 batch.size 的 ProducerBatch，它并不会加入 BufferPool 中，也就不会复用。

之前有个疑问就是：假如 max.request.size 大于 batch.size，那么该条消息会不会分多个 batch 发送到 broker？

答案显然是不会，根据上述所说，如果一个 ProducerRecord 就已经超出了  batch.size 的大小，那么 ProducerBatch 仅包含一个 ProducerRecord，并且该 ProducerBatch 并不会加入到 BufferPool 中。

所以，在 Kafka Producer 调优过程中，根据业务需求，需要特别注意 batch.size 与 max.request.size 之间的大小值的设定，避免内存空间频繁地创建和关闭。