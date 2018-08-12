---
layout: post
title: "Docker实战之搭建Swarm集群"
categories: Docker
tags: Swarm
author: zch
---

* content
{:toc}
之前一直在玩单个 docker 节点，但是如果一家公司需要部署大量 docker 节点，这是远远不够的，我们需要一个集群部署的调用平台，swarm 相比 k8s 、mesos 等等容器集群调度系统，是最容易学的，而且没那么多陌生的概念，简单上手，而且与 docker 无缝链接，毕竟是亲生的嘛。













## 基本概念

- Node

> 一个节点(node)是已加入到 swarm 的 Docker 引擎的实例 当部署应用到集群，你将会提交服务定义到管理节点，接着 Manager 管理节点调度任务到 worker 节点，manager 节点还执行维护集群的状态的编排和群集管理功能，worker 节点接收并执行来自 manager 节点的任务。通常，manager 节点也可以是 worker 节点，worker 节点会报告当前状态给 manager 节点。

![swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm6.png)

- Service

> 服务是要在 worker 节点上要执行任务的定义，它在工作者节点上执行，当你创建服务的时，你需要指定容器镜像。

![swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm7.png)

- Task

> 任务是在 docekr 容器中执行的命令，Manager 节点根据指定数量的任务副本分配任务给 worker 节点。

![swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm8.png)


## 基本指令

1. docker swarm：集群管理，子命令有`init, join, leave, update`。(`docker swarm --help`查看帮助)
2. docker service：服务创建，子命令有`create, inspect, update, remove, tasks`。(`docker service--help`查看帮助)
3. docker node：节点管理，子命令有`accept, promote, demote, inspect, update, tasks, ls, rm`。(`docker node --help`查看帮助)



## 创建集群

由于资源有限，我直接在本地搭建 docker 集群，所以还需要 docker-machine，它可以在本地快速创建虚拟 docker 主机。

首先我们本地增加一个工作节点：

```
$ docker-machine create -d virtualbox worker1
```

这里我们选的是 virtualbox 驱动，如果本地没有，请自行安装：

```
$ brew cask install virtualbox
```

进入 docker 虚拟主机：

```
$ docker-machine ssh worker1
```

将 worker1 主机设定为 swarm 管理节点：

```
$ docker swarm init
```

但是如果你的虚拟主机有两个以上网卡，它就会提示报错：

```
Error response from daemon: could not choose an IP address to advertise since this system has multiple addresses on different interfaces (10.0.2.15 on eth0 and 192.168.99.100 on eth1) - specify one with --advertise-addr
```

所以我们需要选择一个与主机互通的网卡ip：

```
$ docker swarm init --advertise-addr 192.168.99.100

Swarm initialized: current node (gbbyozigwaf8v2bzj6aicpzbn) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-69luztakii9ix7f5osezl0v6l2ibfzp1vqc0gbhcous63hm1fx-8p3vxanj97f2e0jflznihvl8f 192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

这时 wroker1 主机就成为了 swarm 集群管理节点，docker swarm join 这句命令可以使其他 docker 主机



我们来看看节点列表：

```
$ docker node ls
```

![swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm1.png)

发现只有管理节点一个，管理节点也是可以用作工作节点的，也是作为集群的一部分。

接下来创建工作节点：

```
$ docker-machine create -d virtualbox worker2
$ docker-machine create -d virtualbox worker3
$ docker-machine create -d virtualbox worker4
$ docker-machine create -d virtualbox worker5
```

依次创建了4个 docker 节点，我们需要将这几个 docker 节点加入到我们的 swarm 集群，我们通过 docker-machine 进入 docker 虚拟主机，然后输入加入集群的指令：

```
$ docker-machine ssh worker2

$ docker swarm join --token SWMTKN-1-69luztakii9ix7f5osezl0v6l2ibfzp1vqc0gbhcous63hm1fx-8p3vxanj97f2e0jflznihvl8f 192.168.99.100:2377
```

其余 docker 主机都依照这个步骤加入，之后我们再看看集群的节点列表：

```
$ docker node ls
```

![swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm2.png)

发现其余 docker 主机都加入到集群中了，现在我们已经拥有了一个有5个节点的 swarm 集群了，这时我们需要发布 docker 实例到集群中：

```
docker service create --replicas 3 -p 80:80 --name nginx nginx:1.13.7-alpine
```

![swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm5.png)

这就好比我们在单机 docker 中的 docker run 一样方便，只不过现在变成了集群部署，其中 —replicas 指定了 service 有几个实例组成，这几个实例会随机分配到集群的节点上，我们看看 service 都部署到哪些节点上：

```
$ docker service ps nginx
```

![swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm4.png)

这时一个 nginx 的项目就部署到集群中了。

查看集群中有哪些服务：

```
$ docker service ls
```
![swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm3.png)

如果遇到部署的节点太少了，或者我们生产中又购买了服务器，我们需要将项目横向扩展，我们怎么做呢：

```
$ docker service scale nginx=5
```

仅仅一条命令，就可以将 nginx 动态扩展至 5 个实例了。

如果不想某个服务在集群中运行了，可以执行：

```
$ docker service rm servicename
```

这样便可以将服务从集群中删除了，注意这是删除而不是使容器停止哦。

如果想更新某个服务，可以执行：

```
docker service update --image nginx:1.13.8-alpine nginx
```

这样便可以在集群中升级服务的版本了。

参考原文：https://www.jianshu.com/p/881fd566f9f0



