---
layout: post
title: "图解 K8s 核心概念和术语"
categories: K8s
tags: 概念 术语 入门
author: 张乘辉
---

* content
{:toc}
我第一次接触容器编排调度工具是 Docker 自家的 Docker Swarm，主要解决当时公司内部业务项目部署繁琐的问题，我记得当时项目实现容器化之后，花在项目部署运维的时间大大减少了，当时觉得这玩意还挺新鲜的，原来自动化运维可以这么玩。后面由于工作原因，很久没碰过容器方面的知识了。最近在公司的数据同步项目中，需要使用到分布式调度数据同步执行单元，目前使用的方案是将数据同步执行单元打包成镜像，使用 K8s 进行调度，正好趁这个机会了解一下 K8s，下面我就用图解的形式将我所理解的 K8s 分享给大家。









## K8s 三大核心功能

K8s 是一个轻便的和可扩展的开源平台，用于管理容器化应用和服务。通过 K8s 能够进行应用的自动化部署和扩缩容。

K8s 是比容器更上一层的架构，它可以支持多种容器技术，比如我们熟悉的 Docker，K8s 定位是一个容器调度工具，它主要具备以下三大核心能力：

1、自动调度

k8s 将用户部署提交的容器放到 k8s 集群的任意一个节点中，k8s 可以根据容器所需要的资源大小，以及节点的负载情况来决定容器放在哪个节点上面。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823164656.png)



2、自动修复

当 k8s 的健康检查机制发现某个节点出现问题，它会自动将该节点上的资源转移到其它节点上面完成自动恢复。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823170452.png)



3、横向自动扩缩容

在 k8s 1.1+ 版本中，有一个功能叫 “ Horizontal Pod Autoscaler”，简称 “HPA”，意思是 Pod自动扩容，它可以预先定义 Pod 的负载指标，当达到预期设定的负载指标后，就会根据指标自动触发自动动态扩容/缩容行为。

1）横向自动扩容

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823171651.png)

2）横向自动缩容

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823171820.png)





## 节点

从上面的图可以看出来，k8s 集群的节点有两个角色，分别为 Master 节点和 Node 节点，整个 K8s 集群Master 和 Node 节点关系如下图所示：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823181812.png)

1、Master 节点

Master 节点也称为控制节点，每个 k8s 集群都有一个 Master 节点负责整个集群的管理控制，我们上面介绍的 k8s 三大能力都是经过 Master 节点发起的，Master 节点包含了以下几个组件：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823181408.png)

- API Server：提供了 HTTP Rest 接口的服务进程，所有资源对象的增、删、改、查等操作的唯一入口；
- Controller Manager：k8s 集群所有资源对象的自动化控制中心；
- Scheduler：k8s 集群所有资源对象自动化调度控制中心；
- ETCD：k8s 集群注册服务发现中心，可以保存 k8s 集群中所有资源对象的数据。

2、Node

Node 节点的作用是承接 Master 分配的工作负载，它主要有以下几个关键组件：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823184119.png)

- kubelet：负责 Pod 对应容器的创建、启停等操作，与 Master 节点紧密协作；
- kube-porxy：实现 k8s 集群通信与负载均衡的组件。

从图上可看出，在 Node 节点上面，还需要一个容器运行环境，如果使用 Docker 技术栈，则还需要在 Node 节点上面安装 Docker Engine，专门负责该节点容器管理工作。



## Pod

Pod 是 k8s 最重要而且是最基本的一个资源对象，它的结构如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200821153531.png)

从以上 Pod 的结构图可以看出，它其实是容器的一个上层包装结构，这也就是为什么 K8s 可以支持多种容器类型的原因，基于这方面，我理解 k8s 的定位就是一个编排与调度工具，而容器只是它调度的一个资源对象而已。

Pod 可包含多个容器在里面，每个 Pod 至少会有一个 Pause 容器，其它用户定义的容器都共享该 Pause 容器，Pause 容器的主要作用是用于定义 Pod 的 ip 和 volume。

Pod 在 k8s 集群中的位置如下图所示：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823185441.png)



## Label

Label 在 k8s 中是一个非常核心的概念，我们可以将 Label 指定到对应的资源对象中，例如 Node、Pod、Replica Set、Service 等，一个资源可以绑定任意个 Label，k8s 通过 Label 可实现多维度的资源分组管理，后续可通过 Label Selector 查询和筛选拥有某些 Label 的资源对象，例如创建一个 Pod，给定一个 Label，workerid=123，后续可通过 workerid=123 删除拥有该标签的 Pod 资源。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823192435.png)



## Replica Set

Replica Set 目的是为了定义一个期望的场景，比如定义某种 Pod 的副本数量在任意时刻都处于 Peplica Set 期望的值，假设 Replica Set 定义 Pod 的副本数目为：replicas=2，当该 Replica Set 提交给 Master 后，Master 会定期巡检该 Pod 在集群中的数目，如果发现该 Pod 挂掉了一个，Master 就会尝试依据 Replica Set 设置的 Pod 模版创建 Pod，以维持 Pod 的数量与 Replica Set 预期的 Pod 数量相同。

通过 Replica Set，k8s 集群实现了用户应用的高可用性，而且大大减少了运维工作量。因此生产环境一般用 Deployment 或者 Replica Set 去控制 Pod 的生命周期和期望值，而不是直接单独创建 Pod。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823193118.png)

类似 Replica Set 的还有 Deployment，它的内部实现也是通过 Replica Set 实现的，可以说 Deployment 是 Replica Set 的升级版，它们之间的 yaml 配置文件格式大部分都相同。



## Service

Service 是 k8s 能够实现微服务集群的一个非常重要的概念，顾名思义，k8s 的 Service 就是我们平时所提及的微服务架构中的“微服务”，本文上面提及的 Pod、Replica Set 等都是为 Service 服务的资源， 如下图表示 Service、Pod、Replica Set 的关系：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823200632.png)

从上图可看出，Service 定义了一个服务访问的入口，客户端通过这个入口即可访问服务背后的应用集群实例，而 Service 则是通过 Label Selector 实现关联与对接的，Replica Set 保证服务集群资源始终处于期望值。

以上只是一个微服务，通常来说一个应用项目会由多个不同业务能力而又彼此独立的微服务组成，多个微服务间组成了一个强大而又高可用的应用服务集群。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823220527.png)



## Namespace

Namespace 顾名思义是命名空间的意思，在 k8s 中主要用于实现资源隔离的目的，用户可根据不同项目创建不同的 Namespace，通过 k8s 将资源分配到不同 Namespace 中，即可实现不同项目的资源隔离：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200823214534.png)







