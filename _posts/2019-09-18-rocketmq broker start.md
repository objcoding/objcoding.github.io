---
layout: post
title: "RocketMQ Broker启动流程梳理"
categories: RocketMQ
tags: Broker
author: zch
---

* content
{:toc}
Broker 启动的主函数入口：

org.apache.rocketmq.broker.BrokerStartup:

```java
public static void main(String[] args) {
  start(createBrokerController(args));
}
```







1. **创建配置类**

初始化配置主要任务是根据 properties 文件以及命令行参数值，创建了以下配置类：

- nettyServerConfig：封装了作为消息队列服务器的配置信息
- nettyClientConfig：封装了作为NameServer客户端配置信息
- brokerConfig：封装了 Broker 配置信息
- messageStoreConfig：封装了 RocketMQ 存储系统的配置信息



2. **Broker 初始化**

2.1 配置文件加载

- 主题配置加载：

```java
result = result && this.consumerOffsetManager.load();
```

这一步主要是加载 topics.json 文件，并解析生成 TopicConfigSerializerWrapper 对象，并 set 进 topicConfigTable 中。

- 消费者位移管理加载：

```java
result = result && this.subscriptionGroupManager.load();
```

这一步主要是加载 consumerOffset.json 文件，并解析生成 ConsumerOffsetManager 对象，并替换 offsetTable 成员值。

- 消费者订阅组加载：

```java
result = result && this.consumerFilterManager.load();
```

这一步主要是加载 subscriptionGroup.json 文件，并解析生成 SubscriptionGroupManager 对象，并放进 subscriptionGroupTable 中。

- 消费者过滤管理加载：

```java
result = result && this.consumerFilterManager.load();
```

这一步主要是加载 consumerFilter.json 文件，并解析生成 ConsumerFilterManager 对象

- messageStore 消息存储初始化:

```java
if (result) {
  try {
    this.messageStore =
      new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener,
                              this.brokerConfig);
    this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
    //load plugin
    MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
    this.messageStore = MessageStoreFactory.build(context, this.messageStore);
    this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
  } catch (IOException e) {
    result = false;
    e.printStackTrace();
  }
}
```

这一步主要是创建了 DefaultMessageStore 对象，这是 Broker 消息寸处的核心实现，创建该对象时也会启动很多相关服务线程，用于管理 store 的存储。

- messageStore加载：

```java
result = result && this.messageStore.load();
```

1）延迟消息加载：加载 delayOffset.json 文件，解析生成DelayOffsetSerializerWrapper，并加入offsetTable中

2）commitLog加载：MappfileQueue映射文件队列加载，加载定义的storePath目录文件

3）consumeQueue加载



2.2 初始化线程池

- 创建nettyRemotingServer：根据前面初始化好的nettyConfig创建远程通讯服务


- 根据brokerConfig初始化各种线程池：

1）初始化发送消息线程池

2）初始化拉取消息线程池

3）初始化broker管理线程池

4）初始化client管理线程池

5）初始化消费者管理线程池

- 把这些线程池注册到nettyRemotingServer中



2.3 初始化定时任务：

在线程池注册完后，就会开启各种定时任务：

- 开启定时记录 Broker 的状态（消息拉取时间总和、消息发送总和等）

```java
BrokerController.this.getBrokerStats().record();
```

- 消息位移持久化，定时向 consumerOffset.json 文件中写入消费者偏移量

```java
BrokerController.this.consumerOffsetManager.persist();
```

- 消息过滤持久化，定时向 consumerFilter.json 文件写入消费者过滤器信息

```java
BrokerController.this.consumerFilterManager.persist();
```

- 定时禁用消费慢的消费者以保护 Broker，可以设置 disableConsumeIfConsumerReadSlowly 属性，默认 false

```java
BrokerController.this.protectBroker();
```

- 定时打印 Send、Pull、Query、Transaction 信息

```java
BrokerController.this.printWaterMark();
```

- 定时打印已存储在提交日志中但尚未调度到消费队列的字节数

```java
rokerController.this.getMessageStore().dispatchBehindBytes())
```

- 定时获取 namserver 地址

```java
BrokerController.this.brokerOuterAPI.fetchNameServerAddr();
```

如果是从服务器：

- 定时从主服务器获取 TopicConfig、ConsumerOffset、DelayOffset、SubscriptionGroupConfig 等信息

```java
BrokerController.this.slaveSynchronize.syncAll();
```

如果是主服务器：

- 定时打印从服务器落后的字节数

```java
BrokerController.this.printMasterAndSlaveDiff();
```



2.4 添加进程退出时关闭broker资源的钩子函数



3. **Broker 启动**

3.1 messageStore启动：

- 启动各类线程服务：
       1）启动刷盘任务线程
       2）启动commitLog线程
       3）启动存储存储统计服务线程storeStateService
       4）启动延迟定时消息服务线程
       5）启动消息分发到各中Consumer queue服务线程reputMessageService
       6）启动HA主从同步线程
- 启动各类定时任务

3.2 启动netty服务：

remotingServer启动：启动远程通讯服务
fastRemotingServer启动：启动远程通讯服务
broker对外API启动：启动client远程通讯服务

3.3 pullRequestHolderService使拉取消息保持长轮询任务启动

3.4 ClientHouseKeepingService线程定时清除不活动链接任务启动

3.5 过滤服务器任务启动

3.6 向NameServer注册broker信息

3.7 开启定时向NameServer注册broker信息任务



![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rocketmq_13.png)





