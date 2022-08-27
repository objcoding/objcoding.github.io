---
layout: post
title: "Kafka发送消息时提示请求数据过大是怎么回事？"
categories: Kafka
tags: message
author: 张乘辉
---

* content
{:toc}
今天有个小伙伴跟我反馈，在 Kafka 客户端他明明设置了 batch.size 参数，以提高 producer 的吞吐量，但他发现报了如下错误：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/kafka.png)

















然后我去服务器查看了下 producer 的配置，发现没有配置 max.request.size，默认值为 1048576，而他发送的消息大小为 1575543，因此报了这个异常。

然后接下来他跟我讲他已经在客户端配置了 batch.size 的值为 512000，按照这个值的作用，应该是大于这个值才会进行批量发送消息到 broker：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/kafka_2.png)

于是我又得去撸源码，搞清楚 Kafka 发送消息实现细节：

org.apache.kafka.clients.producer.KafkaProducer#doSend:

```java
// ...

// 估算消息的字节大小
int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                                                                   compressionType, serializedKey, serializedValue, headers);

// 确保消息大小不超过发送请求最大值 max.request.size，或者发送缓冲池发小 buffer.memory
ensureValidRecordSize(serializedSize);
long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
// producer callback will make sure to call both 'callback' and interceptor callback
Callback interceptCallback = this.interceptors == null ? callback : new InterceptorCallback<>(callback, this.interceptors, tp);

if (transactionManager != null && transactionManager.isTransactional())
  transactionManager.maybeAddPartitionToTransaction(tp);

// 向 RecordAccumulator 追加消息
RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                                                                 serializedValue, headers, interceptCallback, remainingWaitMs);

// 如果 batch 已经满了，那么就会唤醒sender线程发送批量消息
if (result.batchIsFull || result.newBatchCreated) {
  log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
  this.sender.wakeup();
}
return result.future;
// ...
```

org.apache.kafka.clients.producer.KafkaProducer#ensureValidRecordSize：

```java
private void ensureValidRecordSize(int size) {
  // 如果发送消息的大小比maxRequestSize大，就会抛异常
  if (size > this.maxRequestSize)
    throw new RecordTooLargeException("The message is " + size +
                                      " bytes when serialized which is larger than the maximum request size you have configured with the " +
                                      ProducerConfig.MAX_REQUEST_SIZE_CONFIG +
                                      " configuration.");
  if (size > this.totalMemorySize)
    throw new RecordTooLargeException("The message is " + size +
                                      " bytes when serialized which is larger than the total memory buffer you have configured with the " +
                                      ProducerConfig.BUFFER_MEMORY_CONFIG +
                                      " configuration.");
}
```

从以上源码得出结论，Kafka 会首先判断本次消息大小是否大于 maxRequestSize，如果本次消息大小 maxRequestSize，则直接抛出异常，不会继续执行追加消息到 batch。

batch.size 是 Kafka producer 非常重要的参数，它的值对 Producer 的吞吐量有着非常大的影响，因为我们知道，收集到一批消息再发送到 broker，比每条消息都请求一次 broker，性能会有显著的提高，但 batch.size 设置得非常大又会给机器内存带来极大的压力，因此需要在项目中合理地增减 batch.size 值，才能提高 producer 的吞吐量。

这里来个扩展性的问题：

可能有人会问，如果 producer 发送的消息量非常少，少到不足以填满 batch，因此不足以触发 Sender 线程执行发送消息，那这时怎么办，其实这里还有一个参数与 batch.size 配合使用，叫 linger.ms，这个参数的作用是当达到了 linger.ms 时长后，不管 batch 有没有填满，都会立即发送消息。linger.ms 参数默认值为 0，即默认消息无需批量发送，这时就需要看项目需求来权衡了。









