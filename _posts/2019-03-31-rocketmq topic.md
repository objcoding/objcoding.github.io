---
layout: post
title: "深度解析RocketMQ Topic的创建机制"
categories: RocketMQ
tags: topic
author: 张乘辉
---

* content
{:toc}
我还记得第一次使用rocketmq的时候，需要去控制台预先创建topic，我当时就想为什么要这么设计，于是我决定撸一波源码，带大家从根源上吃透rocketmq topic的创建机制。











topic在rocketmq的设计思想里，是作为同一个业务逻辑消息的组织形式，它仅仅是一个逻辑上的概念，而在一个topic下又包含若干个逻辑队列，即消息队列，消息内容实际是存放在队列中，而队列又存储在broker中，下面我用一张图来说明topic的存储模型：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/rocketmq_4.png)



其实rocketmq中存在两种不同的topic创建方式，一种是我刚刚说的预先创建，另一种是自动创建，下面我开车带大家从源码的角度来详细地解读这两种创建机制。





## 自动创建

默认情况下，topic不用手动创建，当producer进行消息发送时，会从nameserver拉取topic的路由信息，如果topic的路由信息不存在，那么会默认拉取broker启动时默认创建好名为“TBW102”的Topic：

org.apache.rocketmq.common.MixAll：

```java
// Will be created at broker when isAutoCreateTopicEnable
public static final String AUTO_CREATE_TOPIC_KEY_TOPIC = "TBW102";
```

自动创建的开关配置在BrokerConfig中，通过autoCreateTopicEnable字段进行控制，

org.apache.rocketmq.common.BrokerConfig：

```java
@ImportantField
private boolean autoCreateTopicEnable = true;
```

在broker启动时，会调用TopicConfigManager的构造方法，autoCreateTopicEnable打开后，会将“TBW102”保存到topicConfigTable中：

org.apache.rocketmq.broker.topic.TopicConfigManager#TopicConfigManager：

```java
// MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC
if (this.brokerController.getBrokerConfig().isAutoCreateTopicEnable()) {
    String topic = MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC;
    TopicConfig topicConfig = new TopicConfig(topic);
    this.systemTopicList.add(topic);
    topicConfig.setReadQueueNums(this.brokerController.getBrokerConfig()
                                 .getDefaultTopicQueueNums());
    topicConfig.setWriteQueueNums(this.brokerController.getBrokerConfig()
                                  .getDefaultTopicQueueNums());
    int perm = PermName.PERM_INHERIT | PermName.PERM_READ | PermName.PERM_WRITE;
    topicConfig.setPerm(perm);
    this.topicConfigTable.put(topicConfig.getTopicName(), topicConfig);
}
```

broker会通过发送心跳包将topicConfigTable的topic信息发送给nameserver，nameserver将topic信息注册到RouteInfoManager中。

继续看消息发送时是如何从nameserver获取topic的路由信息：

org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#tryToFindTopicPublishInfo：

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
  TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
  if (null == topicPublishInfo || !topicPublishInfo.ok()) {
    this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
    // 生产者第一次发送消息，topic在nameserver中并不存在
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
    topicPublishInfo = this.topicPublishInfoTable.get(topic);
  }

  if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
    return topicPublishInfo;
  } else {
    // 第二次请求会将isDefault=true，开启默认“TBW102”从namerserver获取路由信息
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
    topicPublishInfo = this.topicPublishInfoTable.get(topic);
    return topicPublishInfo;
  }
}
```

如上方法，topic首次发送消息，此时并不能从namserver获取topic的路由信息，那么接下来会进行第二次请求namserver，这时会将isDefault=true，开启默认“TBW102”从namerserver获取路由信息，此时的“TBW102”topic已经被broker默认注册到nameserver了：

org.apache.rocketmq.client.impl.factory.MQClientInstance#updateTopicRouteInfoFromNameServer：

```java
if (isDefault && defaultMQProducer != null) {
  // 使用默认的“TBW102”topic获取路由信息
  topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),1000 * 3);
  if (topicRouteData != null) {
    for (QueueData data : topicRouteData.getQueueDatas()) {
      int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
      data.setReadQueueNums(queueNums);
      data.setWriteQueueNums(queueNums);
    }
  }
}
```

如果isDefault=true并且defaultMQProducer不为空，从nameserver中获取默认路由信息，此时会获取所有已开启自动创建开关的broker的默认“TBW102”topic路由信息，并保存默认的topic消息队列数量。

org.apache.rocketmq.client.impl.factory.MQClientInstance#updateTopicRouteInfoFromNameServer：

```java
TopicRouteData old = this.topicRouteTable.get(topic);
boolean changed = topicRouteDataIsChange(old, topicRouteData);
if (!changed) {
  changed = this.isNeedUpdateTopicRouteInfo(topic);
} else {
  log.info("the topic[{}] route info changed, old[{}] ,new[{}]", topic, old, topicRouteData);
}
```

从本地缓存中取出topic的路由信息，由于topic是第一次发送消息，这时本地并没有该topic的路由信息，所以对比该topic路由信息对比“TBW102”时changed为true，即有变化，进入以下逻辑：

org.apache.rocketmq.client.impl.factory.MQClientInstance#updateTopicRouteInfoFromNameServer：

```java
// Update sub info
{
  Set<MessageQueue> subscribeInfo = topicRouteData2TopicSubscribeInfo(topic, topicRouteData);
  Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
  while (it.hasNext()) {
    Entry<String, MQConsumerInner> entry = it.next();
    MQConsumerInner impl = entry.getValue();
    if (impl != null) {
      impl.updateTopicSubscribeInfo(topic, subscribeInfo);
    }
  }
}
```

**将“TBW102”topic路由信息构建TopicPublishInfo，并将用topic为key，TopicPublishInfo为value更新本地缓存，到这里就明白了，原来broker们千辛万苦创建“TBW102”topic并将其路由信息注册到nameserver，被新来的topic获取后立即用“TBW102”topic的路由信息构建出一个TopicPublishInfo并且据为己有，由于TopicPublishInfo的路由信息时默认“TBW102”topic，因此真正要发送消息的topic也会被负载发送到“TBW102”topic所在的broker中，这里我们可以将其称之为偷梁换柱的做法。**

**当broker接收到消息后，会在msgCheck方法中调用createTopicInSendMessageMethod方法，将topic的信息塞进topicConfigTable缓存中，并且broker会定时发送心跳将topicConfigTable发送给nameserver进行注册。**

自动创建与消息发送时获取topic信息的时序图：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/rocketmq_7.png)





## 预先创建

其实这个叫预先创建似乎更加适合，即预先在broker中创建好topic的相关信息并注册到nameserver中，然后client端发送消息时直接从nameserver中获取topic的路由信息，但是手动创建从动作上来将更加形象通俗易懂，直接告诉你，你的topic信息需要在控制台上自己手动创建。

预先创建需要通过mqadmin提供的topic相关命令进行创建，执行：

```bash
./mqadmin updateTopic
```

官方给出的各项参数如下：

```bash
usage: mqadmin updateTopic [-b <arg>] [-c <arg>] [-h] [-n <arg>] [-o <arg>] [-p <arg>] [-r <arg>] [-s <arg>]
-t <arg> [-u <arg>] [-w <arg>]
-b,--brokerAddr <arg>       create topic to which broker
-c,--clusterName <arg>      create topic to which cluster
-h,--help                   Print help
-n,--namesrvAddr <arg>      Name server address list, eg: 192.168.0.1:9876;192.168.0.2:9876
-o,--order <arg>            set topic's order(true|false
-p,--perm <arg>             set topic's permission(2|4|6), intro[2:W 4:R; 6:RW]
-r,--readQueueNums <arg>    set read queue nums
-s,--hasUnitSub <arg>       has unit sub (true|false
-t,--topic <arg>            topic name
-u,--unit <arg>             is unit topic (true|false
-w,--writeQueueNums <arg>   set write queue nums
```

我们直接定位到其实现类执行命令的方法：

通过broker模式创建：

org.apache.rocketmq.tools.command.topic.UpdateTopicSubCommand#execute：

```java
// -b,--brokerAddr <arg>   create topic to which broker
if (commandLine.hasOption('b')) {
  String addr = commandLine.getOptionValue('b').trim();
  defaultMQAdminExt.start();
  defaultMQAdminExt.createAndUpdateTopicConfig(addr, topicConfig);
  return;
}
```

从commandLine命令行工具获取运行时-b参数重的broker的地址，defaultMQAdminExt是默认的rocketmq控制台执行的API，此时调用start方法，该方法创建了一个mqClientInstance，它封装了netty通信的细节，接着就是最重要的一步，调用createAndUpdateTopicConfig将topic配置信息发送到指定的broker上，完成topic的创建。

通过集群模式创建：

org.apache.rocketmq.tools.command.topic.UpdateTopicSubCommand#execute：

```java
// -c,--clusterName <arg>   create topic to which cluster
else if (commandLine.hasOption('c')) {
  String clusterName = commandLine.getOptionValue('c').trim();
  defaultMQAdminExt.start();
  Set<String> masterSet =
    CommandUtil.fetchMasterAddrByClusterName(defaultMQAdminExt, clusterName);
  for (String addr : masterSet) {
    defaultMQAdminExt.createAndUpdateTopicConfig(addr, topicConfig);
    System.out.printf("create topic to %s success.%n", addr);
  }
  return;
}
```

通过集群模式创建与通过broker模式创建的逻辑大致相同，多了根据集群从nameserver获取集群下所有broker的master地址这个步骤，然后在循环发送topic信息到集群中的每个broker中，这个逻辑跟指定单个broker是一致的。

**这也说明了当用集群模式去创建topic时，集群里面每个broker的queue的数量相同，当用单个broker模式去创建topic时，每个broker的queue数量可以不一致。**

预先创建时序图：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/rocketmq_8.png)



## 何时需要预先创建Topic？

建议线下开启，线上关闭，不是我说的，是官方给出的建议：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/rocketmq_5.png)

rocketmq为什么要这么设计呢？经过一波源码深度解析后，我得到了我想要的答案：

根据上面的源码分析，我们得出，**rocketmq在发送消息时，会先去获取topic的路由信息，如果topic是第一次发送消息，由于nameserver没有topic的路由信息，所以会再次以“TBW102”这个默认topic获取路由信息，假设broker都开启了自动创建开关，那么此时会获取所有broker的路由信息，消息的发送会根据负载算法选择其中一台Broker发送消息，消息到达broker后，发现本地没有该topic，会在创建该topic的信息塞进本地缓存中，同时会将topic路由信息注册到nameserver中，那么这样就会造成一个后果：以后所有该topic的消息，都将发送到这台broker上，如果该topic消息量非常大，会造成某个broker上负载过大，这样消息的存储就达不到负载均衡的目的了。**



