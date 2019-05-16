---
layout: post
title: "由for update引发的血案"
categories: Spring Mybatis Mysql Oracle
tags: autocommit forupdate 数据库事务
author: zch
---

* content
{:toc}
公司的某些业务用到了数据库的悲观锁 for update，但有些同事没有把 for update 放在 Spring 事务中执行，在并发场景下发生了严重的线程阻塞问题，为了把这个问题吃透，秉承着老司机的职业素养，我决定要给同事们一个交代。










## 案发现场

最近公司的某些 Dubbo 服务之间的 RPC 调用过程中，偶然性地发生了若干起严重的超时问题，导致了某些模块不能正常提供服务。我们的数据库用的是 Oracle，经过 DBA 排查，发现了一些 sql 的执行时间特别长，对比发现这些执行时间长的 sql 都带有 for update 悲观锁，于是相关开发人员查看 sql 对应的业务代码，发现 for update 没有放在 Spring 事务中执行，但是按照常理来说，如果 for update 没有加 Spring 事务，每次执行完 Mybatis 都会帮我们 commit 释放掉资源，并发时出现的问题应该是没有锁住对应资源产生脏数据而不是发生阻塞。但是经过代码的调试，不加 Spring 事务并发执行确实会阻塞。



## 案例分析

基于案发现场的问题所在，我特地写了几个针对问题的案例分析测试代码，"talk is cheap, show you the code"：

### 加 Spring 事务执行但不提交事务

```java
public void forupdateByTransaction() throws Exception {
  // 主线程获取独占锁
  reentrantLock.lock();
  
  new Thread(() -> transactionTemplate.execute(transactionStatus -> {
    // select * from forupdate where name = #{name} for update
    this.forupdateMapper.findByName("testforupdate");
    System.out.println("==========for update==========");
    countDownLatch.countDown();
    // 阻塞不让提交事务
    reentrantLock.lock();
    return null;
  })).start();
  
  countDownLatch.await();
  
  System.out.println("==========for update has countdown==========");
  this.forupdateMapper.updateByName("testforupdate");
  System.out.println("==========update success==========");

  reentrantLock.unlock();
}
```

此时 for update 被包装在 Spring 事务中，将事务交由 Spring 管理，根据数据事务机制，sql 执行过程中，只有执行了 commit 或者 rollback 操作， 才会提交事务，所以此时每次执行 commit，for update 没有被释放，会锁住对应资源，直到提交事务释放 for udpate。所以此时的主线程执行更新操作会阻塞。



### 不加 Spring 事务并发执行

```java
public void forupdateByConcurrent() {
  AtomicInteger atomicInteger = new AtomicInteger();

  for (int i = 0; i < 100; i++) {
    new Thread(() -> {
      // select * from forupdate where name = #{name} for update
      this.forupdateMapper.findByName("testforupdate");
      System.out.println("========ok:" + atomicInteger.getAndIncrement());
    }).start();
  }

}
```

首先我们先将数据库连接池的初始化大小调大一点，使该次并发执行至少会获取 2 个以上 ID 不同的 connection 对象来执行 for update，以下是某一次的执行日志：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/transaction.png)

得到测试结果，发现如果有 2 个或以上 ID 不同的 connection 对象执行 sql，会发生阻塞，而 Mysql 不会发生阻塞，至于 Mysql 为什么不会发生阻塞，后面我再给大家解释。

由于我们使用的 druid 连接池，它的 autoCommit 默认为 true，所以我此时将 druid 连接池的 autoCommit 参数设置为 false，再次跑测试代码，发现此时 oracle 不会发生阻塞，我们先记住这个测试结果，下面我会带大家走一波源码，来解释这个现象。

聪明的你可能会想到，Mybatis 的底层源码不是给我们封装了一些重复性操作吗，比如我们执行一条 sql 语句，mybatis 自动为我们 commit 或者 rollback了，这也是 JDBC 框架的基本要求，那么既然 Mybatis 帮我们 commit 了，for update 应该会被释放才对，为什么还会发生阻塞问题呢？如果你能想到这个问题，说明你是个认真思考的人，这个问题我们也是先记住，后面会有解释。



### 加 Spring 事务并发执行

```java
private void forupdateByConcurrentAndTransaction() {
  AtomicInteger atomicInteger = new AtomicInteger();

  for (int i = 0; i < 100; i++) {
    new Thread(() -> transactionTemplate.execute(transactionStatus -> {
      // select * from forupdate where name = #{name} for update
      this.forupdateMapper.findByName("testforupdate");
      System.out.println("========ok:" + atomicInteger.getAndIncrement());
      return null;
    })).start();
  }
}
```

这个案例分析主要是为了测试是否跟 Spring 事务有关联，我将 druid 链接池的 autoCommit 参数分别设置为 true 和 false，发现 for update 在 Spring 事务的包装下并发执行，并不会发生阻塞，从测试结果来看，似乎是跟 Spring 事务有很大的关系。



我们现在总结一下案例分析测试结果：

1. 事务不提交，for update 悲观锁不会被释放；
2. 不加 Spring 事务并发执行 for update 语句，如果有两个以上的不同 ID 的 connection 执行 for update，会发生阻塞现象，Mysql 则不会阻塞；
3. 不加 Spring 事务并发执行 for update 语句，并且 druid 连接池的 autocommit=false，不会发生阻塞；
4. 加 Spring 事务并发执行 for update 语句，不会发生阻塞。

贴上测试代码地址：https://github.com/objcoding/test-forupdate




## 源码走一波

基于上述的案例分析，我们源码走一波，从底层源码的角度来解析为什么会有这样的结果。



### Mybatis 事务管理器

有没有发现，我到现在也是一直在强调 Spring 事务，其实在数据库的角度来说，sql 只要在 START TRANSACTION 与 COMMIT 或者 ROLLBACK 之间执行，就算是一个事务，而我强调的 Spring 事务，指的是在Spring 管理下的事务，而 Mybatis 也有自己的事务管理器，通常我们使用 Mybatis 都是配合 Spring 来使用，而 Spring 整合 Mybatis，在 Mybatis-spring 包中，有一个名叫 SpringManagedTransaction 的类，这个就是 Mybatis 在 Spring 体系下的的 JDBC 事务管理器，Mybatis 用它来管理 JDBC connection 的生命周期，别看它名字是以 Spring 开头，但它和 Spring 的事务管理器没有半毛钱关系。

Mybatis 执行 sql 时会创建一个 SqlSession 会话，关于 SqlSession，坐我旁边的钟同学之前有向我提问过 SqlSession 的创建机制，我特意写了一篇文章，感兴趣的可以看看，这里就不再重复述说了：

「[钟同学，this is for you！](https://mp.weixin.qq.com/s/tTTLDOoqPfqHJLW12Zdo6A)」

在创建 SqlSession 时，相应地会创建一个事务管理器：

org.mybatis.spring.transaction.SpringManagedTransactionFactory#newTransaction：

```java
public Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
  return new SpringManagedTransaction(dataSource);
}
```

创建一个 transaction 时，我们发现传入的 autoCommit 根本没有赋值给 SpringManagedTransaction，这里暗藏玄机，我们继续往下看：

执行 sql 时，Mybatis 会从事务管理器中从数据库连接池中获取一个 connection 对象：

org.mybatis.spring.transaction.SpringManagedTransaction#openConnection：

```java
private void openConnection() throws SQLException {
  this.connection = DataSourceUtils.getConnection(this.dataSource);
  this.autoCommit = this.connection.getAutoCommit();
  this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);
  if (LOGGER.isDebugEnabled()) {
    LOGGER.debug(
      "JDBC Connection ["
      + this.connection
      + "] will"
      + (this.isConnectionTransactional ? " " : " not ")
      + "be managed by Spring");
  }
}
```

这里会从数据库连接池中获取 connection 对象，然后将 connection 对象中的 autoCommit 值赋值给 SpringManagedTransaction！可以这么理解，在 Spring 体系下的 Mybatis 事务管理器，autoCommit 的值被数据库连接池的覆盖掉了！而后面的 debug 日志也说明了，这个 JDBC connection 对象不归你 Spring 管理，我 Mybatis 自己就可以管理了，你 Spring 就别瞎参合了。



sql 执行完之后，Mybatis 会自动帮我们 commit，我们来看 SqlSessionTemplate 的 sqlSession 代理：

org.mybatis.spring.SqlSessionTemplate.SqlSessionInterceptor：

```java
if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
  // force commit even on non-dirty sessions because some databases require
  // a commit/rollback before calling close()
  sqlSession.commit(true);
}
```

判断如果不归 Spring 事务管理，那么会强制执行 commit 操作，我们点进去，发现最终调用的是 Mybatis 的事务管理器的 commit 方法：

org.mybatis.spring.transaction.SpringManagedTransaction#commit：

```java
public void commit() throws SQLException {
  if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Committing JDBC Connection [" + this.connection + "]");
    }
    this.connection.commit();
  }
}
```

问题就出现在这里，前面我也说了，我们使用的 druid 数据库连接池的 autoCommit 默认为 true，而事务管理器获取 connection 对象时，又将 connection 的 autocommit 赋值给事务管理器，如果此时 autoCommit 为 true，Mybatis 认为 connection 已经自动提交事务了，既然这事不归我管，那么我 Mybatis 自然就不会再去 commit 了。

根据测试结果，将 druid 的 autoCommit 设置为 false 后，不会发生阻塞现象，即 Mybaits 会执行下面的 commit 操作。那么问题来了，connection 的 autocommit = true 时，到底有没有 commit ？从测试结果来看，很明显没有 commit。这里就要从数据库层来解释了，由于公司 Oracle 数据库的 autocommit 使用的是默认的 false 值，即需要显式提交 commit 事务才会被提交。这也就是为什么当 druid 的 autoCommit=false 时，并发执行不会产生阻塞现象，因为 Mybatis 已经帮我们自动 commit 了。

而为什么当  druid 的 autoCommit=true 时，Mysql 依然不会阻塞呢？我先开启 Mysql 的日志打印：

```bash
set global general_log = 1;
```

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/transaction_1.png)

查看日志，发现 Mysql 会为每条执行的 sql 设置 autocommit=1，即自动提交事务，无须显式提交 commit，每条 sql 就是一个事务。



### Spring 事务管理器

上面的案例分析中，加了 Spring 事务的并发执行，并不会产生阻塞现象，显然肯定是 Spring 事务做了一些不可描述的动作，Spring 的事务管理器有很多个，这里我们用的是数据库连接池那个管理器，叫  DataSourceTransactionManager，我这里为了灵活控制事务范围的细粒度，用的是声明式事务，我们继续走一波源码，从事务入口一路跟踪进来，发现第一步需要调用 doBegin 方法：

org.springframework.jdbc.datasource.DataSourceTransactionManager#doBegin：

```java
// Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
// so we don't want to do it unnecessarily (for example if we've explicitly
// configured the connection pool to set it already).
if (con.getAutoCommit()) {
  txObject.setMustRestoreAutoCommit(true);
  if (logger.isDebugEnabled()) {
    logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
  }
  con.setAutoCommit(false);
}
```

我们在 doBegin 方法发现了它偷偷地篡改了连接对象 autoCommit 的值，将它设为 false，这里想必大家都会明白其中的原理吧，Spring 管理事务其实就是在 sql 执行前将当前的 connection 对象设置为不自动提交模式，接下来执行的 sql 都不会自动提交，等待事务结束时，Spring 事务管理器会帮我们 commit 提交事务。这也就是为什么加了 Spring 事务的并发执行并不会产生阻塞的原因，原理与上述 Mybatis 所描述的一样。

org.springframework.jdbc.datasource.DataSourceTransactionManager#doCleanupAfterCompletion：

```java
// Reset connection.
Connection con = txObject.getConnectionHolder().getConnection();
try {
  if (txObject.isMustRestoreAutoCommit()) {
    con.setAutoCommit(true);
  }
  DataSourceUtils.resetConnectionAfterTransaction(con, txObject.getPreviousIsolationLevel());
}
catch (Throwable ex) {
  logger.debug("Could not reset JDBC Connection after transaction", ex);
}
```

在事务完成之后，我们还需要将 connection 对象还原，因为 connection 存在于连接池当中，close 时并不会真正关闭，而是被回收回连接池当中了，如果不对 connection 对象进行还原，那么当下一次会话拿到该 connection 对象，autoCommit 还是上一次会话的值，就会产生一些很隐晦的问题。



## 写在最后

其实这个问题从应用层来分析还好，直接撸源码就完了，主要是这个问题还涉及一些数据库底层的一些原理，由于我对数据库还不是那么地专业，所以在这过程中，还特别请教了公司的 DBA 斌哥哥，非常感谢他的协助。

另外，我其实是不太建议使用 for update 这种悲观锁的，它太过于依赖数据库层了，而且当并发量起来了，虽然可以保证数据一致性，但是这样牺牲了性能，会大大影响效率，严重拖垮数据库资源，而且像这次一样，有些开发人员使用了 for update 却忘记 commit 事务了，导致引起很多锁故障。



