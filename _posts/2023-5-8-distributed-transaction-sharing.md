---
layout: post
title: "深度剖析分布式事务，轻松掌握实现原理与应用技巧！"
categories: 分布式事务 Seata
tags: 分布式事务 微服务
author: 张乘辉
---

* content
{:toc}


## 前言

大家好，今天我们来一起探讨分布式事务的相关知识。相信大家都有多多少少接触过分布式事务，因为我们现在写的代码可是服务于亿级用户量级的，那么大的请求量级不可能全部写在一台服务器上面对吧。如果你还没有研究过分布式事务，也没关系，我们今天再一起来探讨一番。我曾经接触过分布式事务相关的中间件框架，比如现在很火的阿里开源的一款分布式事务中间件Seata。目前我在Seata社区主要做一些RPC以及性能优化的相关工作，所以我可能会对分布式事务具体实现比较了解。以Seata为契机，我们一起来探讨分布式事务。



## 什么是事务？

开始前，先来问大家两个问题：

第一问题：什么是事务？

在编写代码的时候，我们常常会遇到各种事务问题。那么，我们该如何清晰明了地描述事务的概念呢？事务是指如何确保对一组（多个）数据操作在执行的过程中，要么全部都能够成功执行，要么全部失败。而且，一旦事务成功执行，所变更的数据不会丢失；若事务失败，所有的数据变更都要回到事务开始之前的状态。简单来说，事务包括多个操作，这些操作要么全部执行，要么全部不执行。

第一问题：保证事务的目的是什么？

在理解事务的概念后，我们需要明确实现事务的最终目的是什么？如果在一组事务中，有些操作执行了，有些没执行，会产生什么问题呢？举个例子，如果你给父母转账1W元，结果你的账户扣了1W元，但是你父母的账户却没有加上1W元，这时你就会开始怀疑自己赚钱的意义。这种情况就是所谓的“数据一致性问题”。

当大家明确了以上两个问题之后，我才能继续往下跟大家继续分享今天的这个主题，因为今天这个主题，都是在围绕着怎么保证事务一致性的问题展开的。

## 单进程下完美的解决方案

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/Xnip2023-04-15_22-44-26.png)

A（原子性）、C（一致性）、I（隔离性）、D（持久性）。C 是事务最终的目标，那么A、I、D 就是为实现这个目标努力的打工仔,如果这几个打工仔不能正常工作的话，那么一致性就得不到保障。

**原子性：**原子性是指一组操作要么全部执行成功，要么全部执行失败。这个概念和事务十分相似。如果不保证原子性，就可能出现在同一个事务中，某些操作执行成功，而另一些操作执行失败的情况，这会导致数据不一致，而且很难恢复。因此，原子性是保障数据一致性的重要特性之一。

**隔离性：** 事务的隔离性指的是多个事务之间的操作不会相互影响，它们之间相互隔离。如果没有隔离性，就好像两个人在同一张画布上画画，一个画猪，一个画狗，最后会画出一个四不像。也就是说，如果不保证隔离性，一个人修改数据时，其他人也可以修改，这会导致数据不一致。

**持久性：**持久性指的是一旦事务提交，所产生的数据变更不会因为任何意外（比如数据库故障或服务器宕机）而丢失。因为如果事务产生的部分数据丢失，就会导致数据不一致。

单机事务实现采用ACID模型，通过加锁实现对需要操作相同数据的事务进行隔离，保证事务之间的操作不会相互影响，从而实现了隔离性。在事务提交之前，记录数据修改前的日志（undo log）和事务需要变更数据的日志（redo log），以保证事务不论在哪个阶段都能通过undo log对事务数据进行回滚，把数据恢复到事务开始之前的状态。同时，通过redo log保证事务在提交后，即使数据库或服务器出现故障，也能重做未成功写入磁盘的数据，实现了事务的持久性和原子性。



## 分布式事务的诞生

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203113.png)

在公司发展初期，由于用户量少、数据量少，系统的并发请求并不高，因此只需要将应用单点部署即可满足业务需求。但随着业务的快速发展和复杂度的增加，几乎每个公司的系统都会从单体架构转向分布式架构，特别是微服务架构。

单进程事务演变成多进程事务时，场景发生了改变。之前是一个人独立完成一项任务，现在变成了多个人协作完成同一项任务。在单进程事务中，决定权在自己手中，因此决定回滚或提交事务较为容易。但在多进程事务中，如何协调多个人的操作以达到一致性，则成为一个难题。因此，需要有一个统一的协调者来协调多个节点的操作，以确保多个进程操作的一致性。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203044.png)

比如从图中看到，假设在RPC调用过程中，其中有一个RPC调用异常了，我们怎么去回滚前面两个已经执行成功的事务呢？

这就不得不涉及到我们应该怎么去设计一个分布式事务的执行模型。

## 分布式事务模型：2PC

目前绝大部分分布式事务框架为 2PC 二阶段事务模型。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203222.png)

2PC协议的核心思路是协调者和参与者通过两个阶段的协商达成最终操作的一致性。首先，第一阶段的目的是确认各个参与者是否具备执行事务的条件。根据第一阶段参与者的响应结果，制定出第二阶段的事务策略。如果第一阶段中任意一个参与者不具备事务执行条件，那么第二阶段的决策就是回滚事务。只有在所有参与者都具备事务执行条件的情况下，才进行整体事务的提交。

但是这个模型也不是万能的，在遇到异常情况，很可能就会造成数据不一致（但是这个不一致，在最后都会有框架驱动达成最终一致性）

我下面举两个例子

**参与者挂掉**

如果在第一阶段，协调者发送Prepare指令给所有的参与者后，参与者挂掉了，那么此时协调者因为迟迟收不到参与者的消息而导致超时，所以协调者在超时之后会统一发送abort指令进行事务回滚。

如果在第二阶段，协调者发送commit或者abort指令给所有参与者后，参与者挂掉了，那么协调者会在超时之后进行消息重发，直到参与者恢复后收到到commit或者abort ，向协调者返回成功。

**协调者挂掉**

协调者在第一阶段发送Prepare指令后挂掉，那么此时参与者此时会一直得不到协调者下一步的指令，那么此时参与者会一直陷入阻塞状态，资源也会一直被锁住，直到协调者恢复之后向参与者发出下一步的指令。

协调者在第二阶段挂掉，那么此时协调者已向所有者发出最后阶段的指令了，所以收到指令的参与者会完成最后的commit或rollback操作，对于参与者来说事务已经结束，所以不存在阻塞和锁的问题， 当协调者恢复后，会把事务日志状态标记为结束。

## CAP 定律

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203307.png)

强一致性的事务一致性方案在单机事务场景下可以完美实现，但在分布式事务场景下效果并不理想。这是因为单机事务和分布式事务所面临的场景不同。在单机事务中，只需要考虑数据一致性问题。而在分布式事务场景中，需要同时考虑数据一致性、多节点的可用性、网络分区等多个问题。因此，强一致性的事务模型始终无法完美解决分布式事务场景。

由此引出CAP定律，什么是CAP定律呢？

CA组合就是保证一致性和可用性，放弃分区容忍性，即不进行分区，不考虑由于网络不通或节点挂掉的问题。那么系统将不是一个标准的分布式系统，我们最常用的关系型数据库就满足了CA。

CP组合就是保证一致性和分区容忍性，放弃可用性。Zookerper就是追求强一致性，放弃了可用性，还有跨行转账，一次转账请求要等待双方银行系统都完成整个事务才能完成。

AP组合就是保证可用性和分区容忍性，放弃一致性。这是分布式系统设计时的选择。

## BASE 理论

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203340.png)

CAP理论表明在分布式系统中，无法同时满足一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance）。在分布式系统中，分区容错性是必须满足的，而可用性是分布式系统设计的主要目标，通常需要牺牲一致性来保证可用性和分区容错性。

但是，牺牲一致性并不意味着完全放弃它。所谓牺牲，是指在一段时间内，系统可以暂时不保证一致性，但最终还是要恢复到一致性状态，通常被称为最终一致性。基于最终一致性模型，BASE理论提出了一套实践理论，从基本可用性、软状态和最终一致性三个方面来指导我们进行分布式系统设计。

## Seata介绍

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203454.png)

Seata（Simple Extensible Autonomous Transaction Architecture，简单可扩展自治事务框架）是 2019 年 1 月份阿里巴巴和蚂蚁集团共同开源的分布式事务解决方案。目前在GitHub已经有超过 2 万+ star，社区非常活跃。我在19年7月份的时候正式加入Seata开源社区。

在整个 Seata 体系下，所有模式（AT、TCC、XA、SAGA）都遵循这套角色模型。

### Seata AT 模式

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203538.png)

从图中以及代码中可以看到，在分布式事务场景下，只需要在发起方的方法上面添加注解@GlobalTransaction注解就可以了，完全不「干扰」业务的逻辑。

### Seata AT 模式：通信交互

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203557.png)

可以看出，AT 模式遵循 TC、TM、RM 交互：

1. 首先TM回向TC服务发送一个Begin指令开启全局事务，TC 返回全局事务xid；
2. 各个分支事务向TC服务发送Branch Register进行分支事务注册；
3. TM 向TC服务决议全局提交或者回滚，TC 收到TM最终的二阶段指令后，会驱动各个分支进行提交或者回滚。

可以看出，Seata AT模式是一个 2PC 事务模型。

### Seata AT 模式如何保证对业务的无入侵？

1、数据源代理

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203625.png)

Seata 在数据源做了一层代理层，所以我们使用 Seata 时，我们使用的数据源实际上用的是 Seata 自带的数据源代理 DataSourceProxy，Seata 在这层代理中加入了很多逻辑，主要是解析 SQL，把业务数据在更新前后的数据镜像组织成回滚日志，并将 undo log 日志插入 undo_log 表中，保证每条更新数据的业务 sql 都有对应的回滚日志存在。

2、一阶段

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203647.png)

可以看出，AT模式的分支事务，必须使用支持ACID的关系型数据且业务与回滚日志需要在同一个数据库中，因为业务SQL和回滚日志，需要使用本地事务同时插入数据库中，要么同时成功要么同时失败。如果分开在不同数据库中，就又会产生分布式事务问题，这纯属于套娃行为了。

3、二阶段提交

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203707.png)

当TM决议全局事务提交，TC会发送commit指令给各个分支事务，因为“业务 SQL”在一阶段已经提交至数据库， 所以 Seata 框架只需将一阶段保存的快照数据和行锁删掉，完成数据清理即可。

4、二阶段回滚

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203735.png)

当TM决议全局事务回滚，TC会发送rollback指令给各个分支事务，回滚方式便是用“before image”还原业务数据；但在还原前要首先要校验脏写，对比“数据库当前业务数据”和 “after image”，如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，出现脏写就需要转人工处理。

从整个流程可以看出来，在没有发生脏写的情况下，所有的事务操作都被Seata数据源代理悄悄地处理了。

### Seata AT 模式：事务隔离级别

刚刚我们说到脏写，那么Seata AT模式是怎么发生脏写或者脏读的呢？这不得不从Seata的默认的事务隔离级别说起。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203823.png)

想象一个场景：

某个全局事务事务下有若干个分支事务，在全局事务执行过程中（全局事务还没执行完），某个本地事务提交了，如果Seata没有采取任何措施，会造成什么问题？

传统意义的脏读是读到了未提交的数据，Seata 脏读是读到了全局事务下未提交的数据，全局事务可能包含多个本地事务，某个本地事务提交了不代表全局事务提交了。

在绝大部分应用在读已提交的隔离级别下工作是没有问题的，而实际上，这当中又有绝大多数的应用场景，实际上工作在读未提交的隔离级别下同样没有问题。

在极端场景下，应用如果需要达到全局的读已提交，Seata 设计了由事务协调器维护的全局写排他锁，来保证事务间的写隔离，同时，将全局事务默认定义在读未提交的隔离级别上。

但是默认情况下，Seata 的全局事务是工作在读未提交隔离级别的，保证绝大多数场景的高效性。

### Seata AT 模式：写隔离

**1、提交成功**

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203919.png)

两个全局事务 tx1 和 tx2，分别对 a 表的 m 字段进行更新操作，m 的初始值 1000。

tx1 先开始，开启本地事务，拿到本地锁，更新操作 m = 1000 - 100 = 900。本地事务提交前，先拿到该记录的 **全局锁** ，本地提交释放本地锁。 tx2 后开始，开启本地事务，拿到本地锁，更新操作 m = 900 - 100 = 800。本地事务提交前，尝试拿该记录的 **全局锁** ，tx1 全局提交前，该记录的全局锁被 tx1 持有，tx2 需要重试等待 **全局锁** 。

tx1 二阶段全局提交，释放 **全局锁** 。tx2 拿到 **全局锁** 提交本地事务。

**2、事务回滚**

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507203948.png)

如果 tx1 的二阶段全局回滚，则 tx1 需要重新获取该数据的本地锁，进行反向补偿的更新操作，实现分支的回滚。

此时，如果 tx2 仍在等待该数据的 **全局锁**，同时持有本地锁，则 tx1 的分支回滚会失败。分支的回滚会一直重试，直到 tx2 的 **全局锁** 等锁超时，放弃 **全局锁** 并回滚本地事务释放本地锁，tx1 的分支回滚最终成功。

因为整个过程 **全局锁** 在 tx1 结束前一直是被 tx1 持有的，所以不会发生 **脏写** 的问题。

### Seata AT 模式：读隔离

Seata AT模式下的脏读是指在全局事务未提交之前，其他业务可能会读取已提交的分支事务的数据。本质上，这意味着Seata默认的全局事务是读未提交。

在特定场景下，可能需要全局读取已提交数据。目前，Seata将通过代理SELECT FOR UPDATE语句来实现此需求。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204153.png)

执行SELECT FOR UPDATE语句将申请全局锁。如果全局锁已被其他事务持有，则Seata将释放本地锁并回滚SELECT FOR UPDATE语句的本地执行，并进行重试。在此过程中，查询将被阻塞，直到全局锁被获取，并确保读取的数据是已提交的，然后才会返回查询结果。

### Seata AT 模式：与XA的区别

seata 的事务提交方式跟 XA 协议的两段式提交在总体上来说基本是一致的，那它们之间有什么不同呢？

我们都知道 XA 协议它依赖的是数据库层面来保障事务的一致性，也即是说 XA 的各个分支事务是在数据库层面上驱动的，由于 XA 的各个分支事务需要有 XA 的驱动程序，一方面会导致数据库与 XA 驱动耦合，另一方面它会导致各个分支的事务资源锁定周期长，这也是它没有在互联网公司流行的重要因素。

前面在将为什么无侵入的时候讲到，Seata 在数据源做了一层代理层，所以我们使用 Seata 时，我们使用的数据源实际上用的是 Seata 自带的数据源代理 DataSourceProxy。

这样做的好处就是，本地事务执行完可以立即释放本地事务锁定的资源，然后向 TC 上报分支状态。

当 TM 决议全局提交时，就不需要同步协调处理了，TC 会异步调度各个 RM 分支事务删除对应的 undo log 日志即可，这个步骤非常快速地可以完成，XA就做不到，它必须同步等待所有分支处理完之后才认为全局事务已完成，这个期间被锁定的资源其它业务是不能访问的，这也就是为什么XA性能这么差的原因。正常的业务来说，二阶段commit的几率远大于rollback，因此Seata AT模式相对于XA性能提升是非常巨大的。

当 TM 决议全局回滚时，RM 收到 TC 发送的回滚请求，RM 通过 XID 找到对应的 undo log 回滚日志，然后执行回滚日志完成回滚操作。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204224.png)

如上图所示，Seata 的 RM 实际上是已中间件的形式放在应用层，不用依赖数据库对协议的支持，完全剥离了分布式事务方案对数据库在协议支持上的要求。

## TCC 模式

TCC是分布式事务的一种解决方案，它也是一种2PC模型。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204245.png)

**TCC优点：**

1、性能高：没有全局锁，本地事务锁在本地操作完成后马上会释放，不会像2PC、3PC 一样整个事务执行的过程都会锁住资源，所以TCC性能非常高。

2、具备隔离性: 通过隔离资源达到事务隔离的目的，先预留资源，再真正使用资源，避免了出现两个事务并发时可能导致的同一个资源被使用多次的问题，适合资源敏感的场景。

3、允许事务失败：可以进行事务回滚。

**TCC缺点：**

1、业务侵入性强： 需要修改原来的结构设计来预留资源， 需要在原有的方法基础上把业务拆分为Try、Confirm、Cancel三个方法。

**TCC适用场景：**

有资源隔离性要求、并且对业务系统有控制权，有修改结构的权限。

### Seata TCC 模式：使用效果

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204306.png)

如图所示，参与者需要实现Try、Confirm、Cancel这三个方法，并在Try方法中添加@TwoPhaseBusinessAction注解，填写二阶段commit和rollback的方式到注解参数中。随后，使用Dubbo等rpc协议发布远程RPC服务，在发起方的方法中添加@GlobalTransactional注解来开启全局事务，然后在全局事务内调用参与者的一阶段Try方法。此时，二阶段就由Seata框架来驱动完成。

### Seata TCC 模式：通信交互

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204328.png)

结合刚刚的使用例子，我们来看看 Seata 是如何实现TCC模式的，在这张通信交互图可以看出，它与AT模式一样遵循 TC、TM、RM 角色模型。

其中TM负责开启全局事务，参与者执行try方法时会注册分支事务，TM决议全局事务提交或回滚后，TC协调者会驱动全局事务内的参与者进行提交或者回滚。

### Seata TCC 模式：实践例子

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204348.png)

如图所示，Try 方法作为一阶段准备方法，需要做资源的检查和预留。在扣钱场景下，Try 要做的事情是就是检查账户余额是否充足，预留转账资金，预留的方式就是冻结 A 账户的 转账资金。Try 方法执行之后，账号 A 余额虽然还是 100，但是其中 30 元已经被冻结了，不能被其他事务使用。

二阶段 Confirm 方法执行真正的扣钱操作。Confirm 会使用 Try 阶段冻结的资金，执行账号扣款。Confirm 方法执行之后，账号 A 在一阶段中冻结的 30 元已经被扣除，账号 A 余额变成 70 元 。

如果二阶段是回滚的话，就需要在 Cancel 方法内释放一阶段 Try 冻结的 30 元，使账号 A 的回到初始状态，100 元全部可用。

用户接入 TCC 模式，最重要的事情就是考虑如何将业务模型拆成 2 阶段，实现成 TCC 的 3 个方法，并且保证 Try 成功 Confirm 一定能成功。相对于 AT 模式，TCC 模式对业务代码有一定的侵入性，但是 TCC 模式无 AT 模式的全局行锁，TCC 性能会比 AT 模式高很多。

### TCC可能会遇到什么样的问题？

即使我们拥有了一套完备的TCC接口，也不能高枕无忧。在微服务架构下，很可能会遇到网络超时、重发、机器宕机等一系列异常情况，这会导致分布式事务执行出现异常。根据蚂蚁多年的实践，我们发现最常见的异常有三种，分别是空回滚、幂等、悬挂。

因此，TCC接口需要解决这三类问题。实际上，Seata框架已经支持这三种异常的处理，我们将把这些异常的处理移植到Seata框架中。这样，业务就无需关注这些异常情况，可以专注于业务逻辑。

虽然业务无需关注这些异常，但了解其内部实现机制有助于更好地排查问题。接下来，我将为大家一一讲解这三类异常出现的原因以及对应的解决方案。

#### Seata TCC 模式：如何防止空回滚？

什么是空回滚？

TCC 服务在未收到 Try 请求的情况下收到 Cancel 请求，这种场景被称为空回滚；空回滚在生产环境经常出现，用户在实现TCC服务时，应允许允许空回滚的执行，即收到空回滚时返回成功。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204430.png)

如图所示，事务协调器在调用 TCC 服务的一阶段 Try 操作时，可能会出现因为丢包而导致的网络超时，此时事务管理器会触发二阶段回滚，调用 TCC 服务的 Cancel 操作，而 Cancel 操作调用未出现超时。

要想防止空回滚，那么必须在 Cancel 方法中识别这是一个空回滚，Seata 是如何做的呢？

Seata 的做法是新增一个 TCC 事务控制表，包含事务的 XID 和 BranchID 信息，在 Try 方法执行时插入一条记录，表示一阶段执行了，执行 Cancel 方法时读取这条记录，如果记录不存在，说明 Try 方法没有执行。

#### Seata TCC 模式：如何防悬挂？

悬挂指的是二阶段 Cancel 方法比 一阶段 Try 方法优先执行，由于允许空回滚的原因，在执行完二阶段 Cancel 方法之后直接空回滚返回成功，此时全局事务已结束，但是由于 Try 方法随后执行，这就会造成一阶段 Try 方法预留的资源永远无法提交和释放了。

那么悬挂是如何产生的呢？

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204600.png)

在图示中，当事务协调器调用TCC服务的一阶段Try操作时，由于网络拥堵等原因，可能会出现超时的情况。此时，事务管理器会触发二阶段回滚，调用TCC服务的Cancel操作，但Cancel调用未超时。之后，被网络拥堵延迟的一阶段Try数据包被TCC服务收到，导致二阶段Cancel请求比一阶段Try请求先执行，这会导致TCC服务在执行晚到的Try之后，永远不会再收到二阶段的Confirm或Cancel请求，从而导致TCC服务悬挂的情况。

用户在实现 TCC 服务时，要允许空回滚，但是要拒绝执行空回滚之后 Try 请求，要避免出现悬挂。

Seata 是怎么处理悬挂的呢？

在 TCC 事务控制表记录状态的字段 status 中增加一个状态：

1. suspended：4

当执行二阶段 Cancel 方法时，如果发现 TCC 事务控制表有相关记录，说明二阶段 Cancel 方法优先一阶段 Try 方法执行，因此插入一条 status=4 状态的记录，当一阶段 Try 方法后面执行时，判断 status=4 ，则说明有二阶段 Cancel 已执行，并返回 false 以阻止一阶段 Try 方法执行成功。

#### Seata TCC 模式：如何幂等控制？

幂等问题指的是 TC 重复进行二阶段提交，因此 Confirm/Cancel 接口需要支持幂等处理，即不会产生资源重复提交或者重复释放。

那么幂等问题是如何产生的呢？

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20230507204639.png)

参与者执行完二阶段之后，由于网络抖动或者宕机问题，会造成 TC 收不到参与者执行二阶段的返回结果，TC 会重复发起调用，直到二阶段执行结果成功。

Seata 是如何处理幂等问题的呢？

同样的也是在 TCC 事务控制表中增加一个记录状态的字段 status，该字段有有 3 个值，分别为：

1. tried：1
2. committed：2
3. rollbacked：3

二阶段 Confirm/Cancel 方法执行后，将状态改为 committed 或 rollbacked 状态。当重复调用二阶段 Confirm/Cancel 方法时，判断事务状态即可解决幂等问题。

## 最后

由于时间和篇幅的限制，只讲了AT模式和TCC模式，而XA和SAGA模式还没有讲到。毕竟分布式事务是一个非常广泛的领域，我也在不断学习中，希望能够在后续分享中给大家带来更多的内容。