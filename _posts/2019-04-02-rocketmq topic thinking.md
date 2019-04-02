---
layout: post
title: "关于RocketMQ Topic的创建机制，我还有一些细节上的思考"
categories: RocketMQ
tags: topic queue
author: zch
---

* content
{:toc}
在撸完RocketMQ Topic的创建机制后，我似乎还有一些意犹未尽的感觉，总觉得还缺一些什么。于是我就趁热打铁，提出以下两点我自己的一些思考。











## 当集群新增master节点后，如何保证队列负载？

假设我现在有两个master broker分别为b1和b2组成了一个集群，我选择手动创建topic1，此时topic1的路由信息会发送到b1和b2，同时b1和b2会将路由信息注册到nameserver，发送topic1的消息时，会从nameserver获取topic1的路由信息，然后进行负载均衡平均分发到b1和b2。

以上是前提。

如果此时topic1的消息量剧增，b1和b2负载过高，集群这时加了b3和b4两个master broker节点，但此时b3和b4并没有topic1的路由信息，也就是topic1此时从nameserver获取到的broker信息只有b1和b2，因此虽然加了b3和b4，但topic1的消息并不会路由到b3和b4去，这时只有新加入的topic才会有机会路由到b3和b4那里。

你们有没有想过是如何处理这个问题呢？或者根本不去处理？

通过撸源码可以知道，RocketMQ目前只能是通过手动配置topic1到b3和b4，那么这时问题又来了，如果集群中有成百上千个topic呢？手动配置真的够呛，流量突然增大这时你手动扩容topic时效性也差。那如何来解决这个问题呢？

**我们可以按业务分集群，把topic归类到不同的集群中，这样每个集群添加broker后，需要重新分配的topic就大大减少了。**

**更好的解决方案是添加一个复制功能，新增的broker自动从nameserver拉取需要复制到新broker的topic配置。期待以后的版本迭代中如愿增加这个功能吧。**





## 如何在集群中固定队列数量？

假设集群有20个master broker，有个topic1，这个topic1需要在集群中保持拥有10个消息队列。

以上是前提。

我们都知道手动创建topic有broker模式创建和集群模式创建，我们可以很简单地通过broker模式来创建topic1拥有10个队列，即broker数量*每个broker队列数量就行了，但是通过broker模式创建的话，就有可能造成某些broker负载过高，于是我想通过集群模式去创建topic，我们都知道集群模式创建broker会默认在集群下的每个broker都创建topic的队列路由信息，那么我现在这个集群中创建的每个topic的消息队列至少都会有20个了。

**有没有可能以后会多一个创建机制：在集群模式下，只需要输入topic名称和消息队列数量，至于队列被分配到哪个broker，取决于broker的负载情况。**

同样期待以后的版本迭代中如愿增加这个功能吧。






