---
layout: post
title: "Kafka 常用运维脚本"
categories: Kafka
tags: shell
author: zch
---

* content
{:toc}










## 集群管理

（1）启动 broker

```bash
$ bin/kafka-server-start.sh daemon <path>/server.properties
```

（2）关闭 broker

```bash
$ bin/kafka-server-stop.sh
```

## topic 管理

kafka-topics.sh 脚本

```bash
# 创建主题
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --partitions 64 --replication-factor 3 --topic test-topic --config xxxxx
# 删除主题（delete.topic.enable=true）
$ bin/kafka-topics.sh --delete --zookeeper localhost:2181 --topic test-topic
# 查询主题列表
$ bin/kafka-topics.sh --zookeeper localhost:2181 --list
# 查询主题详情
$ bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test-topic
# 修改主题
$ bin/kafka-topics.sh --alter --zookeeper localhost:2181 --partitions 64 --topic test-topic
# ...
```

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315145802.png)

## consumer 相关管理

（1）查询消费组

kafka-consumer-groups.sh

```bash
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9200 --group test-group --describe
```

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315150715.png)

（2）重设消费组位移

```bash
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9200 --group test-group --reset-offsets --topic test-topic --to-earliest --execute
```

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315152119.png)

（3）删除消费组

```bash
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9200 --delete --group test-group
```



## topic 分区管理

（1）preferred leader 选举

```bash
$ bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181 --path-to-json-file <path>/preferred-leader-plan.json

# {"partitions":[{"topic":"test-topic","partition":0}]}
```

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315163116.png)

（2）分区重分配

```bash
# 生成分配策略
$ bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --topics-to-move-json-file topics-to-move.json --broker-list "5,6" --generate
# 执行分配策略
$ bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file cluster-reassignment.json --execute
# 验证分配
$ bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file cluster-reassignment.json --verify
# 可通过编写分配策略，增加副本因子 略
```



## Kafka 常见脚本工具

（1）kafka-console-producer.sh

```bash
$ bin/kafka-console-producer.sh --broker-list localhost:9200 --topic test --request-required-acks all --timeout 3000 --message-send-max-retries 3
```

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315164532.png)

（2）kafka-console-consumer.sh

```bash
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9200 --topic test --from-beginning
```

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200315165004.png)

（3）生产者性能测试

```bash
$ bin/kafka-producer-perf-test.sh --topic test-topic-5 --num-records 500000000000 --record-size 200 --throughput 200 --producer-props bootstrap.servers=localhost:9092,localhost:9093,localhost:9094 acks=-1
```

（4）消费者性能测试

```bash
$ bin/kafka-consumer-perf-test.sh --topic-list localhost:9200 --message-size 200 --messages 50000 --topic test-topic
```

（5）查看消息元数据

```bash
$ bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files /dfs5/kafka/data/secLog-2/00000000000110325000.log --print-data-log --deep-iteration > secLog.log
```

（6）获取 topic 当前消息数
```bash
# 获取当前最大位移
$ bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9200 --topic test --time -1
# 当前获取最早位移
$ bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9200 --topic test --time -2
# 以上两个数相减，即可得出 topic 当前在集群的消息总数
```