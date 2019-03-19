---
layout: post
title: "钟同学，this is for you！"
categories: Mybatis Spring
tags: SqlSession transaction
author: zch
---

* content
{:toc}
坐在我旁边的钟同学听说我精通Mybatis源码（我就想不通，是谁透漏了风声），就顺带问了我一个问题：**在同一个方法中，Mybatis多次请求数据库，是否要创建多个SqlSession会话？**

可能最近撸多了，当时脑子里一片模糊，眼神迷离，虽然我当时回答他：**如果多个请求同一个事务中，那么多个请求都在共用一个SqlSession，反之每个请求都会创建一个SqlSession**。这是我们在平常开发中都习以为常的常识了，但我却没有从原理的角度给钟同学分析，导致钟同学茶饭不思，作为老司机的我，感到深深的自责，于是我暗自下定决心，要给钟同学一个交代。











## 不服跑个demo

测试在方法中不加事务时，每个请求是否会创建一个SqlSession：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/mybatis5.png)

从日志可以看出，在没有加事务的情况下，确实是Mapper的每次请求数据库，都会创建一个SqlSession与数据库交互，下面我们再看看加了事务的情况：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/mybatis6.png)

从日志可以看出，在方法中加了事务后，两次请求只创建了一个SqlSession，再次证明了我上面的回答，但是仅仅这样回答是体现完全不出一个老司机应有的职业素养的，所以，我要发车了。



## 什么是SqlSession

在发车之前，我们必须得先搞明白，什么是SqlSession？

简单来说，SqlSession是Mybatis工作的最顶层API会话接口，所有的数据库操作都经由它来实现，由于它就是一个会话，即一个SqlSession应该仅存活于一个业务请求中，也可以说一个SqlSession对应这一次数据库会话，它不是永久存活的，每次访问数据库时都需要创建它。

因此，SqlSession并不是线程安全，每个线程都应该有它自己的 SqlSession 实例，千万不能将一个SqlSession搞成单例形式，或者静态域和实例变量的形式都会导致SqlSession出现事务问题，这也就是为什么多个请求同一个事务中会共用一个SqlSession会话的原因，我们从SqlSession的创建过程来说明这点：

1. 从Configuration配置类中拿到Environment数据源；
2. 从数据源中获取TransactionFactory和DataSource，并创建一个Transaction连接管理对象；
3. 创建Executor对象（SqlSession只是所有操作的门面，真正要干活的是Executor，它封装了底层JDBC所有的操作细节）；
4. 创建SqlSession会话。

每次创建一个SqlSession会话，都会伴随创建一个专属SqlSession的连接管理对象，如果SqlSession共享，就会出现事务问题。



## 从源码的角度分析

源码分析从哪一步作为入口呢？如果是看过我之前写的那几篇关于mybatis的源码分析，我相信你不会在Mybatis源码前磨磨蹭蹭，迟迟找不到入口。

在之前的文章里已经说过了，Mapper的实现类是一个代理，真正执行逻辑的是MapperProxy.invoke()，该方法最终执行的是sqlSessionTemplate。

org.mybatis.spring.SqlSessionTemplate：

```java
private final SqlSession sqlSessionProxy;

public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
                          PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
  notNull(executorType, "Property 'executorType' is required");

  this.sqlSessionFactory = sqlSessionFactory;
  this.executorType = executorType;
  this.exceptionTranslator = exceptionTranslator;
  this.sqlSessionProxy = (SqlSession) newProxyInstance(
    SqlSessionFactory.class.getClassLoader(),
    new Class[] { SqlSession.class },
    new SqlSessionInterceptor());
}
```

这个是创建SqlSessionTemplate的最终构造方法，可以看出sqlSessionTemplate中用到了SqlSession，是SqlSessionInterceptor实现的一个动态代理类，所以我们直接深入要塞：

```java
private class SqlSessionInterceptor implements InvocationHandler {
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    SqlSession sqlSession = getSqlSession(
      SqlSessionTemplate.this.sqlSessionFactory,
      SqlSessionTemplate.this.executorType,
      SqlSessionTemplate.this.exceptionTranslator);
    try {
      Object result = method.invoke(sqlSession, args);
      if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
        // force commit even on non-dirty sessions because some databases require
        // a commit/rollback before calling close()
        sqlSession.commit(true);
      }
      return result;
    } catch (Throwable t) {
      Throwable unwrapped = unwrapThrowable(t);
      if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
        // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
        closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        sqlSession = null;
        Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
        if (translated != null) {
          unwrapped = translated;
        }
      }
      throw unwrapped;
    } finally {
      if (sqlSession != null) {
        closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
      }
    }
  }
}
```

Mapper所有的方法，最终都会用这个方法来处理所有的数据库操作，茶饭不思的钟同学眼神迷离不知道是不是自暴自弃导致撸多了，眼神空洞地望着我，问我spring整合mybatis和mybatis单独使用是否有区别，其实没区别，区别就是spring封装了所有处理细节，你就不用写大量的冗余代码，专注于业务开发。

该动态代理方法主要做了以下处理：

1. 根据当前条件获取一个SqlSession，此时SqlSession可能是新创建的也有可能是获取到上一次请求的SqlSession；
2. 反射执行SqlSession方法，再判断当前会话是否是一个事务，如果是一个事务，则不commit；
3. 如果此时抛出异常，判断如果是PersistenceExceptionTranslator且不为空，那么就关闭当前会话，并且将sqlSession置为空防止finally重复关闭，PersistenceExceptionTranslator是spring定义的数据访问集成层的异常接口；
4. finally无论怎么执行结果如何，只要当前会话不为空，那么就会执行关闭当前会话操作，**关闭当前会话操作又会根据当前会话是否有事务来决定会话是释放还是直接关闭**。



org.mybatis.spring.SqlSessionUtils#getSqlSession：

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
  notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

  SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

  SqlSession session = sessionHolder(executorType, holder);
  if (session != null) {
    return session;
  }

  if (LOGGER.isDebugEnabled()) {
    LOGGER.debug("Creating a new SqlSession");
  }

  session = sessionFactory.openSession(executorType);

  registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

  return session;
}
```

是不是看到了不服跑个demo时看到的日志“Creating a new SqlSession”了，那么证明我直接深入的地方挺准确的，没有丝毫误差。在这个方法当中，首先是从TransactionSynchronizationManager（以下称当前线程事务管理器）获取当前线程threadLocal是否有SqlSessionHolder，如果有就从SqlSessionHolder取出当前SqlSession，如果当前线程threadLocal没有SqlSessionHolder，就从sessionFactory中创建一个SqlSession，具体的创建步骤上面已经说过了，接着注册会话到当前线程threadLocal中。

先来看看当前线程事务管理器的结构：

```java
public abstract class TransactionSynchronizationManager {
  // ...
  // 存储当前线程事务资源，比如Connection、session等
  private static final ThreadLocal<Map<Object, Object>> resources =
    new NamedThreadLocal<>("Transactional resources");
  // 存储当前线程事务同步回调器
  // 当有事务，该字段会被初始化，即激活当前线程事务管理器
  private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
    new NamedThreadLocal<>("Transaction synchronizations");
  // ...
}
```

这是spring的一个当前线程事务管理器，它允许将当前资源存储到当前线程ThreadLocal中，从前面也可看出SqlSessionHolder是保存在resources中。

org.mybatis.spring.SqlSessionUtils#registerSessionHolder：

```java
private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
                                          PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
  SqlSessionHolder holder;
  // 判断当前是否有事务
  if (TransactionSynchronizationManager.isSynchronizationActive()) {
    Environment environment = sessionFactory.getConfiguration().getEnvironment();
    // 判断当前环境配置的事务管理工厂是否是SpringManagedTransactionFactory（默认）
    if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Registering transaction synchronization for SqlSession [" + session + "]");
      }

      holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
      // 绑定当前SqlSessionHolder到线程ThreadLocal中
      TransactionSynchronizationManager.bindResource(sessionFactory, holder);
      // 注册SqlSession同步回调器
      TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
      holder.setSynchronizedWithTransaction(true);
      // 会话使用次数+1
      holder.requested();
    } else {
      if (TransactionSynchronizationManager.getResource(environment.getDataSource()) == null) {
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("SqlSession [" + session + "] was not registered for synchronization because DataSource is not transactional");
        }
      } else {
        throw new TransientDataAccessResourceException(
          "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
      }
    }
  } else {
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
    }
  }
}
```

注册SqlSession到当前线程事务管理器的条件首先是当前环境中有事务，否则不注册，判断是否有事务的条件是synchronizations的ThreadLocal是否为空：

```java
public static boolean isSynchronizationActive() {
  return (synchronizations.get() != null);
}
```

**每当我们开启一个事务，会调用initSynchronization()方法进行初始化synchronizations，以激活当前线程事务管理器。**

```java
public static void initSynchronization() throws IllegalStateException {
  if (isSynchronizationActive()) {
    throw new IllegalStateException("Cannot activate transaction synchronization - already active");
  }
  logger.trace("Initializing transaction synchronization");
  synchronizations.set(new LinkedHashSet<TransactionSynchronization>());
}
```

所以当前有事务时，会注册SqlSession到当前线程ThreadLocal中。

**Mybatis自己也实现了一个自定义的事务同步回调器SqlSessionSynchronization，在注册SqlSession的同时，也会将SqlSessionSynchronization注册到当前线程事务管理器中，它的作用是根据事务的完成状态回调来处理线程资源，即当前如果有事务，那么当每次状态发生时就会回调事务同步器**，具体细节可移步至Spring的org.springframework.transaction.support包。 



回到SqlSessionInterceptor代理类的逻辑，发现判断会话是否需要提交要调用以下方法：

org.mybatis.spring.SqlSessionUtils#isSqlSessionTransactional：

```java
public static boolean isSqlSessionTransactional(SqlSession session, SqlSessionFactory sessionFactory) {
  notNull(session, NO_SQL_SESSION_SPECIFIED);
  notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

  SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

  return (holder != null) && (holder.getSqlSession() == session);
}
```

取决于当前SqlSession是否为空并且判断当前SqlSession是否与ThreadLocal中的SqlSession相等，前面也分析了，如果当前没有事务，SqlSession是不会保存到事务同步管理器的，即没有事务，会话提交。

org.mybatis.spring.SqlSessionUtils#closeSqlSession：

```java
public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
  notNull(session, NO_SQL_SESSION_SPECIFIED);
  notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

  SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
  if ((holder != null) && (holder.getSqlSession() == session)) {
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Releasing transactional SqlSession [" + session + "]");
    }
    holder.released();
  } else {
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Closing non transactional SqlSession [" + session + "]");
    }
    session.close();
  }
}
```

方法无论执行结果如何都需要执行关闭会话逻辑，这里的判断也是判断当前是否有事务，如果SqlSession在事务当中，则减少引用次数，没有真实关闭会话。如果当前会话不存在事务，则直接关闭会话。



## 写在最后

虽说钟同学问了我一个Mybatis的问题，我却中了Spring的圈套，猛然发现整个事务链路都处在Spring的管控当中，**这里涉及到了Spring的自定义事务的一些机制，其中当前线程事务管理器是整个事务的核心与中轴，当前有事务时，会初始化当前线程事务管理器的synchronizations，即激活了当前线程同步管理器，当Mybatis访问数据库会首先从当前线程事务管理器获取SqlSession，如果不存在就会创建一个会话，接着注册会话到当前线程事务管理器中，如果当前有事务，则会话不关闭也不commit，Mybatis还自定义了一个TransactionSynchronization，用于事务每次状态发生时回调处理。**



钟同学，this is for you！


