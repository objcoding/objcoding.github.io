---
layout: post
title: "Docker Overlay网络的一些总结"
categories: Docker
tags: swarm overlay networks
author: 张乘辉
---

* content
{:toc}
在早期的docker版本中，是不支持跨主机通信网络驱动的，也就是说明如果容器部署在不同的节点上面，只能通过暴露端口到宿主机上，再通过宿主机之间进行通信。随着docker swarm集群的推广，docker也有了自家的跨主机通信网络驱动，名叫overlay，overlay网络模型是swarm集群容器间通信的载体，将服务加入到同一个网段上的Overlay网络上，服务与服务之间就能够通信。













## 容器之间可互相通信

overlay网络模型在docker集群节点间的加入了一层虚拟网络，它有独立的虚拟网段，意味着docker容器发送的内容，会先发送到虚拟子网，再由虚拟子网包装为宿主机的真实网址进行发送。

也意味着在overlay网络模型上，docker不会暴露端口给宿主机，所有有关网络的事情都交给overlay去处理了，这样的好处就是在同一台服务器，不会引起端口冲突，理解这点非常重要。

下面我画个模型图给大家理解下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/docker12.png)

在早期使用docker进行微服务部署时，你肯定干过将docker容器所在宿主机的ip作为注册地址，因为如果你将容器内的ip作为注册地址，其他服务将不可访问你的服务，只有一条路，就是将宿主机的ip挂在容器内部处理，或者在容器内部添加ip地址环境变量来处理。

当使用docker swarm进行集群服务部署时，这个问题自然也就随之解决了，当服务已加入到overlay网络中，服务容器内会获得一个overlay网络的一个地址，如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/docker13.png)

而dubbo默认会拿eth0的ip作为注册地址，因此服务只需要将docker容器的ip地址作为注册地址就行了，只要服务与服务之间在同一个ovelay网段下，就可以进行通信。

这里我特别做了一个验证，如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/dubbo_1.png)

![](https://gitee.com/objcoding/md-picture/raw/master/img/dubbo_2.png)

提供者和消费者同时部署了2个实例，在相同overlay网段下，相互之间是可通信的。



## 负载均衡特性

前面我也说了，服务的端口会交给overlay网络去处理。在Overlay网络下，即使该节点并没有部署指定服务，也会监听该服务的端口，docker通过这个特性实现了访问任意一个节点+服务对应端口，就可以访问服务，而且还具备负载均衡特性。

基于该特性，我突发奇想，我把服务的网关加入集群后，无需关心网关被分配在哪个节点上，只需要在集群再上一层nginx层配置若干个节点ip+网关服务端口，就可实现双层负载均衡，即nginx访问网关时的负载均衡，访问集群时的负载均衡。如果此时你正在搭建docker swarm集群，我建议你把网关项目同时加入到集群中，再通过nginx做双层负载均衡。

swarm load balancer：

![](https://gitee.com/objcoding/md-picture/raw/master/img/docker14.png)





## Compose编排文件的网络选择



编排文件有个不太灵活的地方， 初次使用它来创建swarm集群的人可能会犯一些错误，比如说：

1.如果你没有指定网络，它会给你默认创建一个网络，如：

```bash
stackname_default
```

2.假如两个stack栈分别属于不同的overlay网络，那么不同stack栈的服务是不可以进行通信的。

3.如果你指定一个网络，没有写name属性的，那么它会在你指定的网络名字前面加个stackname，这本就不是我们设定好的名字，也不知道docker为什么要这么做。

4.如果该网络已存在，在部署时还会报错，不会智能将该stack服务地加入该网络中。

基于上面几点坑爹的想不明白docker为什么这么做的问题，我建议你在部署服务前，先通过命令创建一个网络：

```bash
$ sudo docker network create -d overlay chaos_net
```

为了防止overlay的网络会跟其它网络有冲突，更严谨的做法是自定义overlay网段：

```bash
$ sudo docker network create -d overlay --subnet=10.0.15.0/24 chaos_net
```



在编排文件的networks上配置defualt属性，在defualt属性下面添加external属性，在其下面填写刚刚生成的网络的名称：

```yaml
networks:
  default:
    external:
      name: chaos_net
```

官方解析如下图所示：

![](https://gitee.com/objcoding/md-picture/raw/master/img/docker15.png)

这么做的好处是可以灵活地将不同stack服务栈加入到相同网络下，也可避免上面提到的几个坑。