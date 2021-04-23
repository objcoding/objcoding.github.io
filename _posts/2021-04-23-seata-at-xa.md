---
layout: post
title: "Seata 分布式事务 XA 与 AT 全面解析"
categories: Seata
tags: Seata
author: FUNKYE
---

* content
{:toc}
Seata 是一款开源的分布式事务解决方案，star高达17300+，社区活跃度极高，致力于在微服务架构下提供高性能和简单易用的分布式事务服务。

注：本期分享借鉴于Seata三位PMC 清铭、煊檍、屹远

分享人：陈健斌（funkye） github id: [a364176773](https://github.com/a364176773)

介绍：同盾科技高级开发工程师 、Seata Committer、Spring cloud alibaba contributor,、Mybatis-Plus contributor(by dynamic-datasource)













## 目录

1. XA模式是什么？
2. 什么是 Seata 的事务模式？
3. AT模式是什么？
4. 为什么Seata要支持XA模式？
5. AT与XA之间的关系；
6. 总结；

## 1.XA模式是什么？

首先正如煊檍兄所言，了解了什么是XA与什么是Seata定义的事务模式，便一目了然。

### 1.1什么是XA

用非常官方的话来说

XA 规范 是 X/Open 组织定义的分布式事务处理（DTP，Distributed Transaction Processing）标准。

XA 规范 描述了全局的事务管理器与局部的资源管理器之间的接口。 XA规范 的目的是允许的多个资源（如数据库，应用服务器，消息队列等）在同一事务中访问，这样可以使 ACID 属性跨越应用程序而保持有效。

XA 规范 使用两阶段提交（2PC，Two-Phase Commit）来保证所有资源同时提交或回滚任何特定的事务。

XA 规范 在上世纪 90 年代初就被提出。目前，几乎所有主流的数据库都对 XA 规范 提供了支持。

### 1.2什么是Seata的事务模式？

> Seata 定义了全局事务的框架。
> 全局事务 定义为若干 分支事务 的整体协调：
> 1.TM 向 TC 请求发起（Begin）、提交（Commit）、回滚（Rollback）全局事务。
> 2.TM 把代表全局事务的 XID 绑定到分支事务上。
> 3.RM 向 TC 注册，把分支事务关联到 XID 代表的全局事务中。
> 4.RM 把分支事务的执行结果上报给 TC。（可选）
> 5.TC 发送分支提交（Branch Commit）或分支回滚（Branch Rollback）命令给 RM。

![img](https://img-blog.csdnimg.cn/20201009170223843.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)

> Seata 的 全局事务 处理过程，分为两个阶段：
> 执行阶段 ：执行 分支事务，并 保证 执行结果满足是 可回滚的（Rollbackable） 和 持久化的（Durable）。
> 完成阶段： 根据 执行阶段 结果形成的决议，应用通过 TM 发出的全局提交或回滚的请求给 TC，
> TC 命令 RM 驱动 分支事务 进行 Commit 或 Rollback。
> Seata 的所谓 事务模式 是指：运行在 Seata 全局事务框架下的 分支事务 的行为模式。
> 准确地讲，应该叫作 分支事务模式。
> 不同的 事务模式 区别在于 分支事务 使用不同的方式达到全局事务两个阶段的目标。
> 即，回答以下两个问题：
> 执行阶段 ：如何执行并 保证 执行结果满足是 可回滚的（Rollbackable） 和 持久化的（Durable）。
> 完成阶段： 收到 TC 的命令后，做到事务的回滚/提交

## 2那么什么是Seata XA 模式？

> XA 模式：
> 在 Seata 定义的分布式事务框架内，利用事务资源（数据库、消息服务等）对 XA 协议的支持，以 XA 协议的机制来管理分支事务的一种 事务模式。
> 执行阶段：
> 可回滚：业务 SQL 操作放在 XA 分支中进行，由资源对 XA 协议的支持来保证 可回滚
> 持久化：XA 分支完成后，执行 XA prepare，同样，由资源对 XA 协议的支持来保证 持久化（即，之后任何意外都不会造成无法回滚的情况）
> 完成阶段：
> 分支提交：执行 XA 分支的 commit
> 分支回滚：执行 XA 分支的 rollback

以下是XA模式在Seata所定义的事务模式下的设计模型
![img](https://img-blog.csdnimg.cn/2020100917074118.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)

### 2.1什么是Seata AT(TXC) 模式？

> 去年 1 月份，Seata 开源了 AT 模式。AT 模式是一种无侵入的分布式事务解决方案。在 AT 模式下，用户只需关注自己的“业务 SQL”，用户的 “业务 SQL” 作为一阶段，Seata 框架会自动生成事务的二阶段提交和回滚操作。

通过简介，其实可以发现AT模式的特点，只需关注自己的业务sql，对业务无入侵的一种分布式事务模式。
那么我们应该知道他是怎么对业务做到无入侵的？

### 2.2AT 模式如何做到对业务的无侵入 ？

#### AT模式一阶段

- 首先，在Seata的组件中，如果你想开启分布式事务，那么就应该在你的业务入口或者事务发起入口加上@GlobalTransactional注解
- 如果你是AT模式就要做好数据源代理（seata1.0后全面支持自动代理），并被sqlsessionfactroy使用（或者直接jdbc操作使用被代理数据源）

可以发现比较关键的异步，与其他模式的区别便是代理数据源，而代理数据源又有什么奥秘呢？

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020100917171591.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)
如上图所示，你的数据源被代理后，通过被DataSourceProxy代理后，你所执行的sql，会被提取，解析，保存前镜像后，再执行业务sql，再保存后镜像，以便与后续出现异常，进行二阶段的回滚操作。

### 2.3 AT 模式如何保证隔离性

首先我们拿到官网所展示的文档来更直观的描述：
![img](https://img-blog.csdnimg.cn/20201009172119860.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)
可以通过上图得出：

一阶段本地事务提交前，需要确保先拿到 全局锁 。

拿不到 全局锁 ，不能提交本地事务。

拿 全局锁 的尝试被限制在一定范围内，超出范围将放弃，并回滚本地事务，释放本地锁。

两个全局事务 tx1 和 tx2，分别对 a 表的 m 字段进行更新操作，m 的初始值 1000。

tx1 先开始，开启本地事务，拿到本地锁，更新操作 m = 1000 - 100 = 900。

本地事务提交前，先拿到该记录的 全局锁 ，本地提交释放本地锁。

tx2 后开始，开启本地事务，拿到本地锁，更新操作 m = 900 - 100 = 800。

本地事务提交前，尝试拿该记录的 全局锁 ，tx1 全局提交前，

该记录的全局锁被 tx1 持有，tx2 需要重试等待 全局锁 ，如tx2等待所超时，那么tx2便回滚本地事务所以他不会产生脏数据。

#### AT 模式二阶段提交

二阶段如果是提交的话，因为“业务 SQL”在一阶段已经提交至数据库，
所以 Seata 框架只需将一阶段保存的快照数据和行锁删掉，完成数据清理即可。

![img](https://img-blog.csdnimg.cn/20201009172347256.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)

#### AT 模式二阶段回滚

二阶段如果是回滚的话，Seata 就需要回滚一阶段已经执行的“业务 SQL”，还原业务数据。
回滚方式便是用“before image”还原业务数据；但在还原前要首先要校验脏写，
对比“数据库当前业务数据”和 “after image”，
如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，
出现脏写就需要转人工处理。

![img](https://img-blog.csdnimg.cn/20201009172434921.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)
完整的AT在Seata所制定的事务模式下的模型图：
![img](https://img-blog.csdnimg.cn/20201009172509465.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)

## 3.为什么支持XA？

首先我们应该从AT去做判断，为什么Seata有了AT模式还去做XA的支持

- 从视角出发：
  首先，我们来总结下AT模式，首先所有的事物发起，都是从TM（不仅AT）
  且数据的读已提交只能在应用中见效（用户自行开发的系统），对资源的查看，无法做到全方面
  而XA可让资源也感知到自身已处于全局事务中，对资源的隔离性可由数据库本身来实现，满足
  全局一致性
- 从入侵性，数据库支持角度：
  业务无入侵的更彻底，少于2个服务的操作，仅使用本地事务即可满足一致性，而AT需要
  全局锁来保证隔离性，所以无论是1个服务，单库的操作，还是n个服务都需要开启全局事务来保证
  隔离性。
  对数据库的支持，如果AT需要支持mysql，pgsql，oracle以外的数据库，需要做适配，并且
  对复杂sql的解析成本更大，开发效率低，支持的sql数量少，XA可全方位支持数据库的sql语句
  多语言支持，如果你有java应用已经使用了seata xa那么本地数据库已经帮我们保证了隔离
  性，即便其余seata不支持的语言和java并行处理下，数据也不会出现不一致的情况。

## 4.为什么Seata要支持XA模式？

- 数据锁定：在整个事务处理过程结束前，涉及数据都被锁定，读写都按隔离级别的定义约束起来。

  AT 模式使用 全局锁 保障基本的 写隔离，实际上也是锁定数据的，只不过锁在 TC 侧集中管理 解锁效率高且没有阻塞的问题，且XA本地数据库可能持有间隙锁，造成锁的粒度更大，锁定更多无辜数据

- 死锁（协议阻塞）： XA prepare 后，分支事务进入阻塞阶段，收到 XA commit 或 XA rollback 前必须阻塞等待。
  如果没有一个靠谱的协调者存在，比如abc三个库的数据被二阶段决议为提交，此时ab收到的指令，提交后，c库在收到指令后挂了，并没有提交xa事务，或者协调者没有做到二阶段重试，那么这个没有提交的xa事务将会一直 持有锁，造成死锁的局面

- 性能差：性能的损耗主要来自两个方面：一方面，事务协调过程，增加单个事务的 RT；另一方面，并发事务数 据的锁冲突，降低吞吐。
  其实主要原因就是上面的阻塞跟数据锁定造成，因为xa的一阶段并非提交，如果一阶段都是提交的场景下，由于At模式的一阶段提交，at的性能是优于xa，因为它锁在tc一侧集中释放，无需多个库进行本地的锁释放

### AT与XA的关系

首先，我们要明确，无论是AT还是XA，他们都是有利用到数据库自带的事务特性，来保证数据一致性和隔离性

比如AT一阶段提交和二阶段回滚，都是执行了本地事务。
比如XA的一阶段和二阶段，也都是利用了数据库本身的事务特性

那么这样一样我们是否应该在数据库层面进行挖掘，AT与XA的关系呢？

首先这个时候，我们肯定要从中找相同，与找不同。AT首当其冲，他有个必须品，也就是undolog表，undolog，相
信了解数据库的同学肯定是知道。
数据库有六种日志分别是：
重做日志（redo log）、回滚日志（undo log）、二进制日志（binlog）、错误日志（errorlog）、
慢查询日志（slow query log）、一般查询日志（general log），中继日志（relay log）

那么数据库的undolog是做什么用的呢？
undolog保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC）

可以发现数据库的undolog跟seata at模式的undolog的作用不谋而合，所以可以判断，at模式的undolog就是把本地事务作用中的undolog，利用他的原理，做到了分布式事务中，来保证了分布式事务下的事务一致性。

那么说完了undolog，redolog呢？

Redolog的作用便是防止在发生故障的时间点，尚有脏页未写入磁盘，在重启mysql服务的时候，根据redo log进行
重做，从而达到事务的持久性这一特性。

那么为什么Seata AT模式没有看到redolog的存在？其实很简单，这个redolog被隐藏的很深，也就是AT模式的一阶段提交，让数据库作为我们的redolog，保证一阶段的数据准确落盘。

这个时候是不是会想到LCN事务模式？他的undolog由数据库来保证，缺少了一个redolog的存在。
其实大可不必思念LCN事务，解析到这里，如果把AT改为一阶段不提交，二阶段提交时，前镜像便是undolog，后镜像便是redolog,也就是说AT其实就是一个不在数据库层面，按照数据库事务思想和实现原理的方式，做到了分布式中的事务一致性。

这时候讲到这里，XA跟AT的关系应该一幕了然了，准确的说，其实应该说是分布式事务跟数据库本地事务的关系，可以说XA的缺点造成了AT模式的出生，锁在多侧（多个库），资源阻塞，性能差。

而AT就像为了把事务的实现决定权从数据库手中，放到了Seata中，自实现sql解析，自实现undolog(redolog)，既然我们没有
办法去直接优化数据库在分布式事务下的问题，那么不如创造一个新的模式，去其糟粕，取其精华。

### Seata AT与XA的优劣

其实上面零零碎碎也说了不少各自的优缺点，现在我们总结一下分3点来做比较

- sql支持
- 隔离性
- 侵入性

我们先讲第一点，由于上面我们总结了，其实AT就是一个自实现的XA事务，所以其实可以知道，AT在sql支持上，是远不及利用本地事务的XA模式，既然AT需要做sql解析，那么背后的实现只能自己来解决，也就是靠Seata社区的贡献者们来贡献解决方案，这是一个长期性的关键问题，但是依然有很多用户选择了重写sql，来获取AT事务模式的支持。
在sql支持上XA无疑是完胜的

第二点隔离性，Seata AT模式通过解析sql获取涉及的主键id，生成行锁，也就是AT模式的隔离就是靠全局锁来保证，粒度细至行级，锁信息存储在Seata-Server一侧。
XA模式的隔离性就是由本地数据库保证，锁存储在各个本地数据库中。
由于XA模式一旦执行了prepare后，再也无法重入这个XA事务，也无法跟其他XA事务共享锁。
因为XA协议，仅是通过XID来start一个xa事务，本身它不存在所谓的分支事务说法，它本事就是一个XA事务而已，也就是说它只管它自己。
这时候可能由同学有疑问了，为什么我在branch_table里看到里XA分支事务呢？其实这个问题根据上面的什么是Seata事务模式可以了解到，Seata的事务模式就是由全局事务，分支事务，锁信息所组成。而XA的分支事务，仅仅是作为一个参与方的存在，也就是说这个XA分支在Seata定义中为分支事务，作为分支信息记录在案，方便宕机后也可以下发二阶段决议信息。
而AT由于锁是自实现，也就相对XA来说，我只要知道用户sql涉及到的数据，是不是数据这个全局事务下的，只要是我默认他就可以使用这个锁，也就解决了重入问题。
我们可以得出总结，XA的隔离性是全局的，AT的隔离性是更灵活且相对全局的(保证所有对数据的写操作被Seata事务覆盖)。
第三点，入侵性，通过我们以上的信息，其实可以发现，谁更底层，谁的入侵性更小，所以由数据库自身所支持的XA模式来说，无疑入侵最小，使用成本最低。

其实说到这里，大家可能会觉得XA模式怎么感觉比AT好这么多，虽然他不支持锁重入，但是我可以避免这个情况发生呀。这时候，我画个图，大家可能会比较理解
![img](https://img-blog.csdnimg.cn/20201013173927439.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)
上图中，右侧图1是at模式运行时，图2时xa模式运行时。可以很明显，xa的阻塞带来的性能下降时非常厉害的，特别是你的分支事务非常多，每个资源的释放必须等到每个分支的数据库去单独释放，后续的事务才能进入。
虽然XA带来的无侵入非常高，但是由于性能下降的程度太大，也就促使了AT的诞生，而现在AT,TCC,SAGA的模式的接受度也越来越高，这也正说明了开发者对性能的要求。
AT可以看作时由Seata社区进行全方面优化，自研的XA模式，最大特点就是解决了XA模式的性能差的问题。
TCC由Seata决定二阶段状态通知，其使用全权交托用户，性能仅仅是2个本地事务+些许rpc开销。
SAGA整个事务链路，事务处理全权交托用户编排，性能完全由用户来保证，Seata作为事务的协助方，记录全局事务的运行状态。
可以看出来，越高入侵性的模式其实背后可优化的点更多，越少入侵性的，也就是会被局限，只能依托组件开发者进行不定期的优化来保证性能。

## 总结

在当前的技术发展中，目前分布式事务就是属于扮演东风的角色，大量的分布式，微服务化，带来的性能提
升非常明显，但是却缺少一个有利的保障，我相信Seata就是承担着这样的一个角色，让万事俱备不欠东风。
Seata项目的最核心的价值在于：

构建一个全面解决分布式事务问题的 标准化 平台。

基于 Seata，上层应用架构可以根据实际场景的需求，灵活选择合适的分布式事务解决方案。

![img](https://img-blog.csdnimg.cn/20201009174207878.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1NzIxMjg3,size_16,color_FFFFFF,t_70#pic_center)