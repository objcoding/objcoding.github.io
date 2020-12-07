---
layout: post
title: "Kafka Producer 异步发送消息居然也会阻塞？"
categories: Kafka
tags: 异步发送
author: 张乘辉
---

* content
{:toc}
Kafka 一直以来都以高吞吐量的特性而家喻户晓，就在上周，在一个性能监控项目中，需要使用到 Kafka 传输海量消息，在这过程中遇到了一个 Kafka Producer 异步发送消息会被阻塞的问题，导致生产端发送耗时很大。

是的，你没听错，Kafka Producer 异步发送消息也会发生阻塞现象，那究竟是怎么回事呢？











在新版的 Kafka Producer 中，设计了一个消息缓冲池，客户端发送的消息都会被存储到缓冲池中，同时 Producer 启动后还会开启一个 Sender 线程，不断地从缓冲池获取消息并将其发送到 Broker，如下图所示：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200912172553.png)

这么看来，Kafka 的所有发送，都可以看作是异步发送了，因此在新版的 Kafka Producer 中废弃掉异步发送的方法了，仅保留了一个 send 方法，同时返回一个 Futrue 对象，需要同步等待发送结果，就使用 Futrue#get 方法阻塞获取发送结果。而我在项目中直接调用 send 方法，为何还会发送阻塞呢？

我们在构建 Kafka Producer 时，会有一个自定义缓冲池大小的参数 `buffer.memory`，默认大小为 32M，因此缓冲池的大小是有限制的，我们不妨想一下，缓冲池内存资源耗尽了会怎么样？

Kafka 源码的注释是非常详细的，RecordAccumulator 类是 Kafka Producer 缓冲池的核心类，而 RecordAccumulator 类就有那么一段注释：

> The accumulator uses a bounded amount of memory and append calls will block when that memory is exhausted, unless this behavior is explicitly disabled.

大概的意思是：

当缓冲池的内存块用完后，消息追加调用将会被阻塞，直到有空闲的内存块。

由于性能监控项目每分钟需要发送几百万条消息，只要 Kafka 集群负载很高或者网络稍有波动，Sender 线程从缓冲池捞取消息的速度赶不上客户端发送的速度，就会造成客户端发送被阻塞。

我写个例子让大家直观感受一下被阻塞的现象：

```java
public static void main(String[] args) {
  Properties properties = new Properties();
  properties.put(ProducerConfig.ACKS_CONFIG, "0");
  properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
  properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.ByteArraySerializer");
  properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092,localhost:9093,localhost:9094");
  properties.put(ProducerConfig.LINGER_MS_CONFIG, 1000);
  properties.put(ProducerConfig.BATCH_SIZE_CONFIG, 1024 * 1024);
  properties.put(ProducerConfig.MAX_REQUEST_SIZE_CONFIG, 5242880);
  properties.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");
  KafkaProducer<String, byte[]> producer = new KafkaProducer<>(properties);
  String str = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
  List<byte[]> bytesList = new ArrayList<>();
  Random random = new Random();
  for (int j = 0; j < 1024; j++) {
    int i1 = random.nextInt(10);
    if (i1 == 0) {
      i1 = 1;
    }
    byte[] bytes = new byte[1024 * i1];
    for (int i = 0; i < bytes.length - 1; i++) {
      bytes[i] = (byte) str.charAt(random.nextInt(62));
    }
    bytesList.add(bytes);
  }

  while (true) {
    long start = System.currentTimeMillis();
    producer.send(new ProducerRecord<>("test_topic", bytesList.get(random.nextInt(1023))));
    long end = System.currentTimeMillis() - start;
    if (end > 100) {
      System.out.println("发送耗时:" + end);
    }
    // Thread.sleep(10);
  }
}
```

以上例子构建了一个  Kafka Producer 对象，同时使用死循环不断地发送消息，这时如果把 `Thread.sleep(10);`注释掉，则会出现发送耗时很长的现象：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200912223722.png)

使用 JProfiler 可以查看到分配内存的地方出现了阻塞：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200912223106.png)

跟踪到源码：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200912223239.png)

发现在 `org.apache.kafka.clients.producer.internals.BufferPool#allocate` 方法中，如果判断缓冲池没有空闲的内存了，则会阻塞内存分配，直到有空闲内存为止。

如果不注释 `Thread.sleep(10);`这段代码则不会发生阻塞现象，打断点到阻塞的地方，也不会被 Debug 到，从现象能够得知，`Thread.sleep(10);`使得发送消息的频率变低了，此时 Sender 线程发送的速度超过了客户端的发送速度，缓冲池一直处于未满状态，因此不会产生阻塞现象。

除了以上缓冲池内存满了会发生阻塞之外，Kafka Produer 其它情况都不会发生阻塞了吗？非也，其实还有一个地方，也会发生阻塞！

Kafka Producer 通常在第一次发送消息之前，需要获取该主题的元数据 Metadata，Metadata 内容包括了主题相关分区 Leader 所在节点信息、副本所在节点信息、ISR 列表等，Kafka Producer 获取 Metadata 后，便会根据 Metadata 内容将消息发送到指定的分区 Leader 上，整个获取流程大致如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200912190702.png)

如上图所示，Kafka Producer 在发送消息之前，会检查主题的 Metadata 是否需要更新，如果需要更新，则会唤醒 Sender 线程并发送 Metatadata 更新请求，此时 Kafka Producer 主线程则会阻塞等待 Metadata 的更新。

如果 Metadata 一直无法更新，则会导致客户端一直阻塞在那里。



