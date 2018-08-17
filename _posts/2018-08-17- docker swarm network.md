---
layout: post
title: "Docker网络模型"
categories: Docker
tags: Swarm network
author: zch
---

* content
{:toc}
单机容器内的通信是通过 docker 自带的网桥连接互通的，如果是集群，那么做这些单机网络模型就行不通了，因为集群必然会将一个服务的多个任务需要分布到不同的机器上进行部署，因为如果把所有的任务都分配到一台机器部署了，这台服务器有故障，就会导致这个服务不能正常运行了，而且一个集群内，不同主机之间的容器是怎么进行通信的呢，这里我们就涉及到 docker 网络模型。













## 容器的端口映射

还记得 docker 单机部署时的 run -p port:port 吗？这样做的目的是将 docker 容器内的端口映射到宿主机的端口上，以便能够通过外网 ip 访问到 docker 容器，这时我们就想，如果我们把所有容器的接口都暴露在宿主机中，通过访问外网 ip 来达到容器间通信，这不是万事大吉了吗？

但是在集群中如果这样暴露容器的端口，是有问题的，如果其中一个容器监听了宿主机 8080 端口，那么其他容器只能映射到其它端口了，因为端口并不能被共享，而且映射到宿主机的端口上，意味着容器也就暴露到外网了，如果要限制访问，那么就需要做一些安全配置。



## 单机网络模型

在介绍跨主机网络模型前，先来看看单机网络模型，在安装 docker 之后，docker就会有4种网络模型，分别是：

- host 模式，使用 --net=host 指定。
- container 模式，使用 --net=container:NAME_or_ID 指定。
- none 模式，使用 --net=none 指定。
- bridge 模式，使用 --net=bridge 指定，默认设置。

但这四种网络模式都仅限于单机，其中 bridge 网络模型是 docker 的默认单机网络模型，它会将一个主机上的 docker 容器连接到一个虚拟网桥上，这个虚拟桥名称为 docker0，如下图：

![brige](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/d_network1.png)

单机中的容器之间就可以通过 docker0 互相通信了，但是如果容器被分布在不同主机上，在没有跨主机网络模型前，只能通过映射端口的形式来通信了。

![brige](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/d_network2.png)

如上图，net1 和 net2 都代表一台主机中的 docker0 网络，在同主机下的容器通过 docker0 网络互相通信，但是在不同主机中却又是隔离的。



## 跨主机网络模型

docker 1.9 版本之后，加入了一个默认的 overlay 的网络模型，它是 docker swarm 内置的跨主机通信方案，这是一个基于vxlan协议的网络实现，其作用是虚拟出一个子网，让处于不同主机的容器能透明地使用这个子网。所以跨主机的容器通信就变成了在同一个子网下的容器通信，看上去就像是同一主机下的bridge网络通信。

















## docker network









