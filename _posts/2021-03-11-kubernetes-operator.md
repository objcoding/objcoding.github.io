---
layout: post
title: "中通缓存服务平台基于 Kubernetes Operator 的服务化实践"
categories: Kubernetes
tags: Operator zcache
author: 张乘辉
---

* content
{:toc}
ZCache 是中通下一代缓存服务平台，实现多种缓存类型自动部署，提供 Proxy 访问层，通过 Proxy 层提供指令限制、访问权限、限流、分片处理等功能，通过自研 K8s Operator 实现自动部署与故障转移，实现集群的高可用，提供完善统计、监控、运维功能、减少运维成本和误操作，提高机器的利用率，提供灵活的伸缩性，方便用户接入缓存服务。











## 背景

当前公司的缓存使用搜狐 TV 开源的 CacheCloud 缓存服务平台进行托管，CacheCloud 可以快速在不同机器上部署一套 Redis 集群，在用户层，CacheCloud 将每个集群抽象成一个“应用”，用户可以在”应用“中很方便地对自己申请的集群进行查看和命令执行、申请扩容等等。

随着公司的业务不断发展，在使用 CacheCloud 过程中伴随而来的问题也接踵而至：

1、资源隔离问题

由于 CacheCloud 所管理的物理机器是集群共享的，这样可以有效地利用机器资源，因此用户间的集群节点很可能会共享同一个物理机，且没有对资源进行隔离，比如某个集群访问量高会影响另一个集群等。

2、集群访问权限粒度问题

用户申请一个应用，即拥有了一个完整的 Redis 集群资源，用户对该集群拥有很大的权限，且不好对集群权限进行很好的管理。

3、集群资源不均衡

由于用户每申请一个应用，就会创建一个完整的 Redis 集群，该集群初始容量为 8G，但在实际使用过程中，用户仅使用了 2G 缓存资源，这个问题在使用 CacheCloud 过程中普遍存在于每个应用中，缓存资源使用率非常低。

4、仅支持 Redis 类型的集群

仅仅支持 Redis 缓存类型无法满足公司业务的告诉发展，某些业务需要存储大 Key 大 Value，这种类型也许使用 HBase 作缓存会更加合适。

5、集群节点无法做到高可用性

如果集群某个节点挂了，仅能通过 Redis 的高可用对故障的节点进行迁移，后续还需要运维介入，将挂掉的节点重启。



## ZCache 设计思想

基于以上的几个问题，我们知道目前 CacheCloud 的各种不足之处，它基于集群托管化管理的思想不足以应对公司日益增长的业务需求，我们需要设计一个全新的缓存服务平台，该平台需要解决以上遇到的问题，经过一系列调研，我总结以下几个设计要点：

1、提供 Proxy 代理层

用户不再是通过直连的形式与集群进行交互，在用户与集群之间，增加一层 Proxy 层，通过 Proxy 层，可以将 Proxy 的服务与缓存集群隔离开，因为用户只与 Proxy 层交互，不再通过直连的形式与缓存集群进行交互，有效地避免了网络拥堵对其它集群的影响，同时也减少了缓存集群的 TCP 连接数，而且 Proxy 是一个无状态的服务，理论上可以对 Proxy 的服务无限水平扩容。通过 Proxy 层，我们可以做很多事情，比如指令限制，访问权限控制，限流等等，同时还可以对 Key 做分片处理。由于用户不再需要直连集群，因此用户不再需要关心缓存类型。

2、使用 Kubernetes Operator 进行自动化部署

通过 Proxy 代理层，已经解决了大部分的问题了，那么缓存的实例应该怎么进行部署呢？如果还是像 CacheCloud 那样直接在物理机器上面进行搭建，则无法解决集群节点的高可用性，无法做到自动维护集群节点之间的稳定关系，如果使用 K8s 进行集群的部署，则可以利用 K8s Operator 的高可用特性，维持集群节点间的高可用性，这时集群可以利用 Redis Sentinel 可以进行主从切换做到故障转移，通过 K8s Operator 实现集群节点的高可用性，利用这两层高可用机制维持了集群的稳定性，同时减少运维的工作量。

ZCache 整体架构如下图所示：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20210309112105.png)



## 为什么要使用 Operator

K8s 所托管的容器，一般都是无状态的，这非常适合微服务的理念，使用 K8s 对微服务进行自动部署和编排，再利用 K8s Deployment 特性，能够维护服务节点的数量，提高微服务架构的稳定性，且还能够根据流量的大小，对微服务的实例进行扩/缩容处理。

但如果利用 K8s 对有状态的服务进行部署，就不是那么好处理了，比如使用 K8s 部署 MySql、RocketMQ 集群、Redis 集群等等，对于这些有状态实例，删除/添加实例，其他节点直接都需要做相关的配置变更与状态维护，而且这些操作，在以往我们都是通过人工操作进行的，如果使用 K8s 部署，就失去了它引以为傲的自动化特性。

K8s 也意识到这点不足，在 1.5 版本中引入了 StatefulSet 资源，StatefulSet 主要是用于处理有状态的容器编排，它能够为容器提供一些列有状态的标识，比如指定的 volume、网络标识、容器编号索引等等，再结合 K8s 的 Headless Service，就能够实现对容器的拓扑状态和存储状态的维护。

经过一系列的对 StatefulSet 实践，我发现想要维护 Redis Sentinel 集群的拓扑关系，会异常困难，由于 Pod 重启后 IP 会变化，因此我们需要编写复杂的脚本来维护它们之间的关系，通过自定义的脚本来识别集群之间各个节点之间的拓扑关系，而且这个过程中往往是非常复杂的，而且很容易出错。

既然 K8s StatefulSet 资源都不能很好地对有状态服务进行管理，还有哪些方法可以弥补 StatefulSet 的不足呢？在 2016 年，CoreOS 引入了 Opoerator 的思想，用来扩充 K8s 管理有状态应用的能力。



## Operator 原理

在 k8s 官网上面是这么介绍 Operator 的：

> Operator 是 Kubernetes 的扩展软件，它利用定制资源管理应用及其组件。Operator 遵循 Kubernetes 的理念，特别是在控制器方面。

官方的描述虽然简单，却概括了 Operator 核心原理，我们可以捉重点：定制资源、控制器。下面我用自己的理解尽量通俗地讲解这两个重点。

1、定制资源

在以往我们使用 K8s，会在 yaml 文件（或者通过 API 编写）上定义好各种资源的信息，比如部署实例个数 replicas、镜像名称等等，将定义好的资源提交到 K8s 之后，K8s 会负责维护你这些资源的状态，比如实例少了，会根据定义好的资源信息，将实例维持在 replicas 数量，最终确保集群维护的资源状态于定义的资源一致。

至于资源格式，K8s 已经帮我们定义好了，比如 Deployment 资源，我们只需要按照 Deployment 的资源格式填写并提交到 K8s 中，即可定义一个 Deployment 资源。

同理我们也可定义属于自己的资源类型，在 K8s 中叫作 “CRD”，全称 “CustomResourceDefinition”，举个例子，我们需要自定义一个名为 “zcacheclusters.com.zto.zcache” 的 CRD 资源，如下所示：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: zcacheclusters.com.zto.zcache
spec:
  group: com.zto.zcache
  scope: Namespaced
  names:
    plural: zcacheclusters
    singular: zcachecluster
    kind: ZcacheCluster
    shortNames:
# ... ...
```

2、控制器

在 K8s 中已经有了很多自带的控制器，比如 Deployment、StatefulSet 等等，举个 Deployment 的例子，将某个服务实例的 Deployment 资源定义好，其中 replicas=2，提交到 K8s 集群之后，Deployment 控制器会根据定义的资源，创建两个服务实例的 Pod，并且无限循环地监听集群中服务实例的状态，当服务有变化时，会不断协调最后确保整个集群的服务与定义的一致为止。

同理，Operator 在 K8s 的角色也是一个控制器，根据用户自定义的 CRD 资源，我们可以实现一个针对这个 CRD 资源的控制器，在 Operator 控制器内部，可以调用 K8s API 的客户端，用于实现复杂的控制逻辑，也就是说，以往我们需要调用 K8s API 处理各种逻辑， Operator 控制器将这些操作封装成一个自定义的控制器，我们只需要将自定义的 CRD 资源提交到 K8s 中，即可处理该 CRD 资源。

无论是 K8s 的自带控制，还是我们自定义的控制，它们都把高级的指令集封装成一个个控制器，将其转换为低级操作。

Operator 就是一个自定义的控制器，用户可以随意编写自己想要的控制器功能，由此可见，Operator 几乎可执行任何操作：扩展复杂的应用，应用版本升级，极大地扩展了 K8s 的能力，同时减轻了开发人员的负担。



## ZCache Operator

在讲 Zcache Operator 的实现之前，我先让大家了解下 ZCache Redis 缓存实例的部署架构，ZCache 参考了 Codis 的架构思想，ZCache 的 Redis 底层缓存实例是一组组的 Redis 主从架构，理论上可无限扩展主从的数量，对于用户来说，可以认为 ZCache 是一个无限容量的缓存服务。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20210311143200.png)

我们部署一个 Redis 哨兵集群，通常是先部署 Sentinel 节点，再部署一个主从加入 Sentinel，一个 Sentinel 集群可添加多个主从，我们后续可以继续往 Sentinel 添加主从。

ZCache 的 Operator 也需要满足这个部署顺序，当 ZCache 需要扩容时，会往 Sentinel 添加若干组主从，同时 Operator 需要维护哨兵集群中 Sentinel 节点与主从之间的关系。

下面表示 ZCache 创建一组名为 group-1 主从redis 的定制资源到控制器监听资源状态的处理过程：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20210311135306.png)

提前在 K8s 中自定义了名为 ZcacheCluster 的 CRD 资源，用户编写 ZcacheCluster 的资源，从以上流程图可知，用户目的是为了创建两个 Redis Pod 实例，并且将其维护为一组名为 group-1 的主从，K8s 不断 Watch 该组资源的状态。

以上流程图也表达了控制器的核心理念，控制器的逻辑就是封装了 K8s API 的处理逻辑过程，将它们抽象成一个个控制器的模式，使用户只需要定义相关控制器的资源并提交到 K8s 中，即可完成资源的托管与状态维护。

由于 ZCache 是基于 Java 编写的，官方提供的 operator-sdk 是 Go 语言编写的，如果要自己实现一个套 Java 版的  operator-sdk 成本太大，我在 GitHub 中找到了一个非常不错的 Java 版本 operator-sdk，ZCache Operator 决定基于 java-operator-sdk 编写，从 GitHub 的 Feature 看， java-operator-sdk 的主要特性如下：

1. 处理 K8s API 事件；
2. 自定义资源监视的自动注册；
3. 失败时重试操作；
4. 智能事件调度（仅处理同一资源的最新事件）。

 具体的实现可查看 GitHub 地址：https://github.com/java-operator-sdk/java-operator-sdk

ZCache Operator 的每次触发创建和更新动作，都会调用 io.javaoperatorsdk.operator.api.ResourceController#createOrUpdateResource 接口，ZCache Operator 具体实现逻辑如下：

```java
@Override
public UpdateControl<ZcacheResource> createOrUpdateResource(ZcacheResource zcacheResource, Context<ZcacheResource> context) {
  logger.info("create or update resource:{}, retryNumber:{}, lastAttempt:{}",
              zcacheResource, context.retryInfo().getRetryNumber(), context.retryInfo().isLastAttempt());
  // 一、创建或更新资源（sentinel、redis-ms）
  // 部署缓存资源通常来说，会先创建一组 sentinel 节点，接着按需创建一组组 m - s
  // 因此：
  // 1、一个 operator 只存在一组 sentinel 节点（默认 3 个），存在多组 m - s
  // 2、sentinel 节点单独创建 statefulSet，每次 m - s 单独创建 statefulSet（replicas = 2）
  // 注：创建的资源都是没有任何状态的（即不形成 m - s 主从，sentinel 不监控 master 节点）
  UpdateControl<ZcacheResource> result = generateHandler.handle(zcacheResource);
  if (!result.isUpdateCustomResource()) {
    // 二、检查资源并给定对应状态
    // 1、检查资源 redis 、sentinel 节点数量
    // 2、资源状态设定
    // 2.1 redis 资源 master 节点选定，另外一个节点则监听 master 节点，组成主从架构
    // 2.2 每个 sentinel 节点分别 monitor master 节点
    result = checkHandler.handle(zcacheResource);
    if (!result.isUpdateCustomResource()) {
      // 三、更新资源配置（获取资源 podIP，通过创建redis-client客户端，调用 redis api 设置）
      // 1、更新 redis 节点配置
      // 2、更新 sentinel 节点配置
      return configHandler.handle(zcacheResource);
    }
  }
  return result;
}
```

我将 ZCache Operator 监听逻辑目前分成三部分：

1、创建或更新资源

GenerateHandler 处理器的逻辑主要是使用 K8s API 创建缓存资源，比如：Redis StatefulSet、Redis Service、Sentinel HeadlessService、Sentinel StatefulSet 等，用这些资源搭建哨兵集群。

*在这个过程中，还有一个小插曲，以上的资源创建好之后，如果有变更并不会触发 Operator，仅仅只是它们自身控制器在维护其状态而已，经过深入看 java-operator-sdk 的相关源码以及机制，已经搞明白为啥通过Operator 创建的的默认资源（deployment、statefulset 等等）不触发 Operator 控制器了，这些默认的资源需要在 Operator 内部对这些资源进行 Watch 并处理才行，本质上来说 java-operator-sdk 也是将用户所写的 controller当成一个资源进行 Watch 而已。因此，通过 Operator 创建的这些资源，一定要记得调用 Watch 监听。*

2、检查资源并给定对应状态

这部分是 ZCache Operator 最为关键的逻辑，它体现了 Operator 的核心关键能力：资源状态维护。

文章前面提到，如果使用 K8s StatefulSet 部署有状态的集群服务，想要维护集群的拓扑关系与状态，会异常困难，需要编写一系列复杂的脚本去处理。通过自定义 Operator，我们就可以在实现中添加集群服务拓扑关系与状态的维护逻辑了。

3、更新资源配置

如果我们需要变更集群的配置，通常来说我们需要停止集群节点，并进行滚动更新，运维成本极大，ZCache Operator 的设计理念之一，就是尽可能减少人工运维的成本，因此 ZCache Operator 还能支持集群服务的配置更新，运维仅需要在 ZCache 控制台简单操作，即可完成集群配置的更新。

在 ZCache Operator 的设计中，我把相关的处理逻辑抽象成一个个处理器，随着业务的发展，以后增加处理其只需要实现 Handler 接口即可。

以上是  ZCache Operator 的整体设计理念，接下来，我们要想，我们如何将编写好的 Operator 部署到 K8s 集群中呢？

Operator 部署并没有严格的要求，只要 Operator 能够访问 K8s 集群，以及能够被 K8s 触发执行即可，最简单的做法就是将 Operator 作为 K8s 集群中的一个 Pod，为了 Operator 高可用，使用 Deployment 进行部署，同时将 Replicas 设置为 "1" 即可。





