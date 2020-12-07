---
layout: post
title: "搭建RocketMQ集群"
categories: RocketMQ
tags: rocketmq
author: 张乘辉
---

* content
{:toc}
记录一下搭建RocketMQ集群的过程.











## RocketMQ部署类型

### 单个Master

单机模式, 即只有一个Broker, 如果Broker宕机了, 会导致RocketMQ服务不可用, 不推荐使用.



### 多Master模式

组成一个集群, 集群每个节点都是Master节点, 配置简单, 性能也是最高, 某节点宕机重启不会影响RocketMQ服务, 缺点就是如果某个节点宕机了, 会导致该节点未被消费的消息在在节点恢复前不可订阅.



### 多Master多Slave模式，异步复制

每个Master配置一个Slave, 多对Master-Slave, Master与Slave消息采用异步复制方式, 主从消息一致会有毫秒级的延迟. 优点是弥补了多Master模式下节点宕机后在恢复前不可订阅的问题, 在Master宕机后, 消费者还可以从Slave节点进行消费, 缺点就是如果Master宕机, 磁盘损坏的情况下, 如果没有即使将消息复制到Slave, 会导致有少量消息丢失.



### 多Master多Slave模式，同步双写

每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，主备都写成功，向应用返回成功。数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高。缺点就是性能比异步复制模式略低，大约低10%左右，发送单个消息的RT会略高。





## 集群的一些概念

![rocketmq-cluster](https://gitee.com/objcoding/md-picture/raw/master/img/rocketmq_cluster.png)

- Name Server: 是一个几乎无状态节点, 可集群部署, 节点之间间无任何信息同步. 
- Broker:  Broker分为Master与Slave, 一个Master可以对应多个Slave, 但是一个Slave只能对应一个Master, Master与Slave的对应关系通过指定相同的BrokerName, 不同的BrokerId来定义, BrokerId为0表示Master, 非0表示Slave. Master也可以部署多个. 每个Broker与Name Server集群中的所有节点建立长连接, 定时注册Topic信息到所有Name Server. 
- Producer: producer与Name Server集群中的其中一个节点（随机选择）建立长连接, 定期从Name Server取Topic路由信息, 并向提供Topic服务的Master建立长连接, 且定时向Master发送心跳. Producer完全无状态, 可集群部署.
- Comsumer: Consumer与Name Server集群中的其中一个节点（随机选择）建立长连接, 定期从Name Server 取Topic路由信息, 并向提供Topic服务的Master、Slave建立长连接, 且定时向Master、Slave发送心跳. Consumer既可以从Master订阅消息, 也可以从Slave订阅消息, 订阅规则由Broker配置决定. 

*以上概念来源于RocketMQ开发手册*



## 搭建多 Master多Slave 异步复制模式

- 下载:

```bash
$ wget http://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.3.2/rocketmq-all-4.3.2-source-release.zip
```

- 解压:

```bash
$ unzip rocketmq-all-4.3.2-source-release.zip
$ mv rocketmq-all-4.3.2 rocketmq 
```

- Maven编译:

```bash
$ mvn -Prelease-all -DskipTests clean install -U
```

- 修改broker配置文件:


Master1

```bash
$ vim ${rocketmq_home}/conf/2m-2s-sync/broker-a.properties

brokerClusterName=test_hk_hk3_hd_mq_rocket_1
brokerName=broker-a
# 主服务器必须为0
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
#nameServer地址，分号分割
namesrvAddr=172.17.4.60:9876;172.17.4.61:9876
```

Slave1
```bash
$ vim ${rocketmq_home}/conf/2m-2s-sync/broker-a-s.properties

brokerClusterName=test_hk_hk3_hd_mq_rocket_1
brokerName=broker-a
# 从服务器必须大于0
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
#nameServer地址，分号分割
namesrvAddr=172.17.4.60:9876;172.17.4.61:9876
```
其余另一对Master-Slave与此类推.



另外再列出具体的配置信息与注释: 
```bash
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a|broker-b
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口，10911为默认值
listenPort=10911
#表示Master监听Slave请求的端口,默认为服务端口+1
haListenPort=10912
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER  异步复制Master
#- SYNC_MASTER  同步双写Master
#- SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH  异步刷盘
#- SYNC_FLUSH  同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```



- 启动:
进入rocketMQ解压目录下的bin文件夹
启动nameServer:
```bash
$ nohup sh bin/mqnamesrv &
```
启动broker:
```bash
$ nohup sh mqbroker -c broker-a.properties >/dev/null 2>&1 &
$ nohup sh mqbroker -c broker-a-s.properties >/dev/null 2>&1 &
$ nohup sh mqbroker -c broker-b.properties >/dev/null 2>&1 &
$ nohup sh mqbroker -c broker-b-s.properties >/dev/null 2>&1 &
```
如果运行出现如下错误:
"The broker[broker-a, 172.17.4.60:10911] boot success. serializeType=JSON and name server is 172.17.4.60:9876"
"INFO: os::commit_memory(0x00000006c0000000, 2147483648, 0) failed; error='Cannot allocate memory' (errno=12)"

那么修改一下runserver.sh和runbroker.sh的合适的jvm参数就可以了


- 搭建控制台
本地下载源码:
```bash
$ git clone https://github.com/apache/rocketmq-externals
```
配置好配置文件后, 打包:
```bash
$ mvn clean package
```
将打好的jar包上传到服务器

服务器运行:
```bash
$ nohup java -jar rocketmq-console-ng-1.0.0.jar 2>&1 &
$ tail -f nohup.out
```













