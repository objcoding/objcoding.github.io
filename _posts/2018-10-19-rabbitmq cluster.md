---
layout: post
title: "RabbitMQ集群原理与部署"
categories: RabbitMQ
tags: 消息队列 中间件 集群
author: zch
---

* content
{:toc}
在项目中想要 RabbitMQ 变得更加健壮，就要使得其变成高可用，所以我们要搭建一个 RabbitMQ 集群，这样你可以从任何一台 RabbitMQ 故障中得以幸免，并且应用程序能够持续运行而不会发生阻塞。而 RabbitMQ 本身是基于 Erlang 编写的，Erlang 天生支持分布式（通过同步 Erlang 集群各节点的 cookie 来实现），因此不需要像 ActiveMQ、Kafka 那样通过 ZooKeeper 分别来实现 HA 方案和保存集群的元数据。















## 集群架构

### 元数据

RabbitMQ 内部有各种基础构件，包括队列、交换器、绑定、虚拟主机等，他们组成了 AMQP 协议消息通信的基础，而这些构件以元数据的形式存在，它始终记录在 RabbitMQ 内部，它们分别是：

- 队列元数据：队列名称和它们的属性



- 交换器元数据：交换器名称、类型和属性



- 绑定元数据：一张简单的表格展示了如何将消息路由到队列



- vhost元数据：为 vhost 内的队列、交换器和绑定提供命名空间和安全属性

在单一节点上，RabbitMQ 会将上述元数据存储到内存上，如果是磁盘节点（下面会讲），还会存储到磁盘上。





### 集群中的队列

这里有个问题需要思考，RabbitMQ 默认会将消息冗余到所有节点上吗？这样听起来正符合高可用的特性，只要集群上还有一个节点存活，那么就可以继续进行消息通信，但这也随之为 RabbitMQ 带来了致命的缺点：

1. 每次发布消息，都要把它扩散到所有节点上，而且对于磁盘节点来说，每一条消息都会触发磁盘活动，这会导致整个集群内性能负载急剧拉升。
2. 如果每个节点都有所有队列的完整内容，那么添加节点不会给你带来额外的存储空间，也会带来木桶效应，举个例子，如果集群内有个节点存储了 3G 队列内容，那么在另外一个只有 1G 存储空间的节点上，就会造成内存空间不足的情况，也就是无法通过集群节点的扩容提高消息积压能力。

解决这个问题就是通过集群中唯一节点来负责任何特定队列，只有该节点才会受队列大小的影响，其它节点如果接收到该队列消息，那么就要根据元数据信息，传递给队列所有者节点（也就是说其它节点上只存储了特定队列所有者节点的指针）。这样一来，就可以通过在集群内增加节点，存储更多的队列数据。





### 分布交换器

交换器其实是我们想象出来的，它本质是一张查询表，里面包括了交换器名称和一个队列的绑定列表，当你将消息发布到交换器中，实际上是你所在的信道将消息上的路由键与交换器的绑定列表进行匹配，然后将消息路由出去。有了这个机制，那么在所有节点上传递交换器消息将简单很多，而 RabbitMQ 所做的事情就是把交换器拷贝到所有节点上，因此每个节点上的每条信道都可以访问完整的交换器了。



![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rabbit_mq_11.jpg)



### 内存节点与磁盘节点

关于上面队列所说的问题与解决办法，又有了一个伴随而来的问题出现：如果特定队列的所有者节点发生了故障，那么该节点上的队列和关联的绑定都会消失吗？

1. 如果是内存节点，那么附加在该节点上的队列和其关联的绑定都会丢失，并且消费者可以重新连接集群并重新创建队列；
2. 如果是磁盘节点，重新恢复故障后，该队列又可以进行传输数据了，并且在恢复故障磁盘节点之前，不能在其它节点上让消费者重新连到集群并重新创建队列，如果消费者继续在其它节点上声明该队列，会得到一个 404 NOT_FOUND 错误，这样确保了当故障节点恢复后加入集群，该节点上的队列消息不回丢失，也避免了队列会在一个节点以上出现冗余的问题。

接下来说说内存节点与磁盘节点在集群中的作用，在集群中的每个节点，要么是内存节点，要么是磁盘节点，如果是内存节点，会将所有的元数据信息仅存储到内存中，而磁盘节点则不仅会将所有元数据存储到内存上， 还会将其持久化到磁盘。

在单节点 RabbitMQ 上，仅允许该节点是磁盘节点，这样确保了节点发生故障或重启节点之后，所有关于系统的配置与元数据信息都会重磁盘上恢复；而在 RabbitMQ 集群上，允许节点上至少有一个磁盘节点，在内存节点上，意味着队列和交换器声明之类的操作会更加快速。原因是这些操作会将其元数据同步到所有节点上，对于内存节点，将需要同步的元数据写进内存就行了，但对于磁盘节点，意味着还需要及其消耗性能的磁盘写入操作。

RabbitMQ 集群只要求至少有一个磁盘节点，这是有道理的，当其它内存节点发生故障或离开集群，只需要通知至少一个磁盘节点进行元数据的更新，如果是碰巧唯一的磁盘节点也发生故障了，集群可以继续路由消息，但是不可以做以下操作了：

- 创建队列
- 创建交换器
- 创建绑定
- 添加用户
- 更改权限
- 添加或删除集群节点

这是因为上述操作都需要持久化到磁盘节点上，以便内存节点恢复故障可以从磁盘节点上恢复元数据，解决办法是在集群添加 2 台以上的磁盘节点，这样其中一台发生故障了，集群仍然可以保持运行，且能够在任何时候保存元数据变更。





## 集群部署

### 设置主机名

由于 RabbitMQ 集群连接是通过主机名来连接服务的，必须保证各个主机名之间可以 ping 通，所以需要做以下操作：

修改hostname：

```bash
$ hostname node1 # 修改节点1的主机名
$ hostname node2 # 修改节点2的主机名
```

编辑`/etc/hosts`文件，添加到在三台机器的`/etc/hosts`中以下内容：

```bash
$ sudo vim /etc/hosts

193.xxx.61.78 node1
111.xxx.254.127 node2
```



### 复制 Erlang cookie

这里将 node1 的该文件复制到 node2，由于这个文件权限是 400 为方便传输，先修改权限，非必须操作，所以需要先修改 node2 中的该文件权限为 777

```bash
$ chmod 777 /var/lib/rabbitmq/.erlang.cookie
```

用 scp 拷贝到节点 2

```bash
$ scp /var/lib/rabbitmq/.erlang.cookie node2:/var/lib/rabbitmq/
```

将权限改回来

```bash
$ chmod 400 /var/lib/rabbitmq/.erlang.cookie
```



### 组成集群

在节点 2 执行如下命令：

```bash
$ rabbitmqctl stop_app # 停止rabbitmq服务
$ rabbitmqctl reset # 清空节点状态
$ rabbitmqctl join_cluster rabbit@node1 # node2和node1构成集群,node2必须能通过node1的主机名ping通
$ rabbitmqctl start_app  # 开启rabbitmq服务
```

在任意一台机上面查看集群状态：

```bash
$ rabbitmqctl cluster_status
```

```
Cluster status of node rabbit@node1 ...
[{nodes,[{disc,[rabbit@node1,rabbit@node2]}]},
 {running_nodes,[rabbit@node2,rabbit@node1]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]}]
...done.
```

第一行是集群中的节点成员，disc 表示这些都是磁盘节点，第二行是正在运行的节点成员。

登陆管理后台查看节点状态：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rabbit_mq_8.png)

cluster 搭建起来后若在 web 管理工具中 rabbitmq_management 的 Overview 的 Nodes 部分看到 “Node statistics not available” 的信息，说明在该节点上 web 管理插件还未启用。 执行如下命令：

```bash
$ rabbitmq-plugins enable rabbitmq_management
```

重启 RabbitMQ：

```bash
$ rabbitmqctl stop
$ rabbitmq-server -detached
```



### 设置内存节点

如果节点需要设置成内存节点，则加入集群的命令如下：

```bash
$ rabbitmqctl join_cluster --ram rabbit@node1
```

其中`–ram`指的是作为内存节点，如果不加，那就默认为内存节点。

如果节点在集群中已经是磁盘节点了，通过以下命令可以将节点改成内存节点：

```bash
$ rabbitmqctl stop_app  # 停止rabbitmq服务
$ rabbitmqctl change_cluster_node_type ram # 更改节点为内存节点
$ rabbitmqctl start_app # 开启rabbitmq服务
```

现在再查看集群状态：

```bash
$ rabbitmqctl cluster_status

Cluster status of node rabbit@node1 ...
[{nodes,[{disc,[rabbit@node1]},{ram,[rabbit@node2]}]},
 {running_nodes,[rabbit@node2,rabbit@node1]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]}]
...done.
```

可以看到，节点 2 已经成为内存节点了。



### 镜像队列

当节点发生故障时，尽管所有元数据信息都可以从磁盘节点上将元数据拷贝到本节点上，但是队列的消息内容就不行了，这样就会导致消息的丢失，那是因为在默认情况下，队列只会保存在其中一个节点上，我们在将集群队列时也说过。

聪明的 RabbitMQ 早就意识到这个问题了，在 2.6以后的版本中增加了，队列冗余选项：镜像队列。镜像队列的主队列（master）依然是仅存在于一个节点上，其余从主队列拷贝的队列叫从队列（slave）。如果主队列没有发生故障，那么其工作流程依然跟普通队列一样，生产者和消费者不会感知其变化，当发布消息时，依然是路由到主队列中，而主队列通过类似广播的机制，将消息扩散同步至其余从队列中，这就有点像 fanout 交换器一样。而消费者依然是从主队列中读取消息。

一旦主队列发生故障，集群就会从最老的一个从队列选举为新的主队列，这也就实现了队列的高可用了，但我们切记不要滥用这个机制，在上面也说了，队列的冗余操作会导致不能通过扩展节点增加存储空间，而且会造成性能瓶颈。

命令格式如下：

```bash
$ rabbitmqctl set_policy [-p Vhost] Name Pattern Definition [Priority]

-p Vhost: 可选参数，针对指定vhost下的queue进行设置
Name: policy的名称
Pattern: queue的匹配模式(正则表达式)
Definition: 镜像定义，包括三个部分ha-mode, ha-params, ha-sync-mode
    ha-mode: 指明镜像队列的模式，有效值为 all/exactly/nodes
        all: 表示在集群中所有的节点上进行镜像
        exactly: 表示在指定个数的节点上进行镜像，节点的个数由ha-params指定
        nodes: 表示在指定的节点上进行镜像，节点名称通过ha-params指定
    ha-params: ha-mode模式需要用到的参数
    ha-sync-mode: 进行队列中消息的同步方式，有效值为automatic和manual
priority: 可选参数，policy的优先级
```

举几个例子：

- 以下示例声明名为ha-all的策略，它与名称以"ha"开头的队列相匹配，并将镜像配置到集群中的所有节点：

```bash
$ rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

上述命令会将所有的队列冗余到所有节点上，一般可以拿来测试。

- 策略的名称以"two"开始的队列镜像到群集中的任意两个节点，并进行自动同步：

```bash
$ rabbitmqctl set_policy ha-two "^two." '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

- 以"node"开头的队列镜像到集群中的特定节点的策略：

```bash
$ rabbitmqctl set_policy ha-nodes "^nodes." '{"ha-mode":"nodes","ha-params":["rabbit@nodeA", "rabbit@nodeB"]}'
```



![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rabbit_mq_12.png)





## 集群的负载均衡

HAProxy 提供高可用性、负载均衡以及基于 TCP 和 HTTP 应用的代理，支持虚拟主机，它是免费、快速并且可靠的一种解决方案。

集群负载均和架构图：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rabbit_mq_10.png)



安装 HAProxy：

```bash
$ yum install haproxy
```

编辑 HAProxy 配置文件：

```bash
$ vim /etc/haproxy/haproxy.cfg
```

加入以下内容：

```bash
#绑定配置
listen rabbitmq_cluster 0.0.0.0:5670
    #配置TCP模式
    mode tcp
    #加权轮询
    balance roundrobin
    #RabbitMQ集群节点配置
    server node1 193.112.61.178:5672 check inter 2000 rise 2 fall 3
    server node2 111.230.254.127:5672 check inter 2000 rise 2 fall 3

#haproxy监控页面地址
listen monitor 0.0.0.0:8100
    mode http
    option httplog
    stats enable
    stats uri /stats
    stats refresh 5s
```

启动 HAProxy：

```bash
$ /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg
```



后台管理：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/rabbit_mq_9.png)






