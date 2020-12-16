---
layout: post
title: "Mybatis+Spring整合,没有考虑Interceptor线程安全,却只在debug时诱发bug"
categories: Mybatis Spring
tags: mybatis spring 整合 bug interceptor 线程安全
author: 李文龙
---

* content
{:toc}
> 本文原创来自我部门框架组核心开发李文龙



先看下发现这个bug的一个背景，但背景中的问题，并非这个bug导致：

最近业务部门的一位开发同事找过来说，自己在使用公司的框架向数据库新增数据时，新增的数据被莫名其妙的回滚了，并且本地开发环境能够复现这个问题。公司的框架是基于SpringBoot+Mybatis整合实现，按道理这么多项目已经在使用了， 如果是bug那么早就应该出现问题。我的第一想法是不是他的业务逻辑有啥异常导致事务回滚了，但是也并没有出现什么明显的异常，并且新增的数据在数据库中是可以看到的。于是猜测有定时任务在删数据。询问了这位同事，得到的答案**却是否定的**。没有办法，既然能本地复现那便是最好解决了，决定在本地开发环境跟源码找问题。
刚开始调试时只设置了几个断点，代码执行流程一切正常，查看数据库中新增的数据也确实存在，但是当代码全部执行完成后，数据库中的数据却不存在了，程序也没有任何异常。继续深入断点调试，经过十几轮的断点调试发现偶尔会出现``org.apache.ibatis.executor.ExecutorException: Executor was closed.``，但是程序跳过一些断点时，就一切正常。在经过n轮调试未果之后，还是怀疑数据库有定时任务或者数据库有问题。于是重新创建一个测试库新增数据，这次数据新增一切正常，此时还是满心欢喜，至少已经定位出问题的大致原因了，赶紧找了DBA帮忙查询是否有SQL在删数据，果然证实了自己的想法。后来让这位开发同事再次确认是否在开发环境的机器上有定时任务有删除数据的服务。这次尽然告诉我**确实有定时任务删数据**，问题得以解决，原来他是新接手这个项目，对项目不是很熟悉，真的。。。。。。

现在我们回到标题重点**没有考虑Interceptor线程安全,导致断点调试时才会出现的bug**
晚上下班后，突然想到调试中遇到的``org.apache.ibatis.executor.ExecutorException: Executor was closed.``是啥情况？难道这地方还真的是有bug？马上双十一到了，这要是在双十一时整个大bug，那问题可大了。第二天上班后，决定要深入研究一下这个问题。由于不知道是什么情况下才能触发这个异常，只能还是一步一步断点调试。
首先看实现的Mybatis拦截器，主要代码如下：

```java
@Intercepts({
        @Signature(method = "query", type = Executor.class, args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(method = "query", type = Executor.class, args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),
        @Signature(method = "update", type = Executor.class, args = {MappedStatement.class, Object.class})
})
public class MybatisExecutorInterceptor implements Interceptor {

    private static final String DB_URL = "DB_URL";

    private Executor target;

    private ConcurrentHashMap<Object, Object> cache = new ConcurrentHashMap<>();

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object proceed = invocation.proceed();
        //Executor executor = (Executor) invocation.getTarget();
        Transaction transaction = target.getTransaction();
        if (cache.get(DB_URL) != null) {
            //其他逻辑处理
            System.out.println(cache.get(DB_URL));
        } else if (transaction instanceof SpringManagedTransaction) {
            Field dataSourceField = SpringManagedTransaction.class.getDeclaredField("dataSource");
            ReflectionUtils.makeAccessible(dataSourceField);
            DataSource dataSource = (DataSource) ReflectionUtils.getField(dataSourceField, transaction);
            String dbUrl = dataSource.getConnection().getMetaData().getURL();
            cache.put(DB_URL, dbUrl);
            //其他逻辑处理
            System.out.println(cache.get(DB_URL));
        }
        //其他逻辑略...
        return proceed;
    }

    @Override
    public Object plugin(Object target) {
        if (target instanceof Executor) {
            this.target = (Executor) target;
            return Plugin.wrap(target, this);
        }
        return target;
    }
}
```

调试过程中，一步步断点，便会出现如下异常：

```java
Caused by: org.apache.ibatis.executor.ExecutorException: Executor was closed.
	at org.apache.ibatis.executor.BaseExecutor.getTransaction(BaseExecutor.java:78)
	at org.apache.ibatis.executor.CachingExecutor.getTransaction(CachingExecutor.java:51)
	at com.bruce.integration.mybatis.plugin.MybatisExecutorInterceptor.intercept(MybatisExecutorInterceptor.java:37)
	at org.apache.ibatis.plugin.Plugin.invoke(Plugin.java:61)
```

根据异常信息,将代码定位到了org.apache.ibatis.executor.BaseExecutor.getTransaction() 方法

```java
 @Override
  public Transaction getTransaction() {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    return transaction;
  }
```

发现当变量``closed``为true时会抛出异常。那么只要定位到修改``closed``变量值的方法不就知道了。通过idea工具的搜索只找到了一个修改该变量值的地方。那就是``org.apache.ibatis.executor.BaseExecutor#close()``方法

```java
@Override
  public void close(boolean forceRollback) {
    try {
      ....省略
    } catch (SQLException e) {
      // Ignore. There's nothing that can be done at this point.
      log.warn("Unexpected exception on closing transaction.  Cause: " + e);
    } finally {
      ....省略
      closed = true; //只有该处修改为true
    }
  }
```

于是将断点添加到finally代码块中，看看什么时候会走到这个方法。当一步步debug时，发现还没有走到close方法时，closed的值已经被修改为true，又抛出了Executor was closed.异常。奇怪了？难道还有其他代码会反射修改这个变量，按道理Mybatis要是修改自己代码中的变量值，不至于用这种方式啊，太不优雅了，还增加代码复杂度。

没办法，又是经过n次一步步的断点调试。终于偶然的发现在idea debug窗口显示出这样的提示信息。

```
Skipped breakpoint at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor:423 because it happened inside debugger evaluation
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200901220354970.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyMDIyMzg=,size_16,color_FFFFFF,t_70#pic_center)
从提示上看，不过是跳过了某个断点而已，其实之前就已经注意到这个提示，但是这次怀着好奇搜索了下解决方案。
原来idea在展示类的成员变量，或者方法参数时会调用对象的toString()，怀着试试看的心态，去掉了idea中的toString选项。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200901223830384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyMDIyMzg=,size_16,color_FFFFFF,t_70#pic_center)
**再次断点调试，这次竟然不再出现异常**，原来是idea显示变量时调用对象的toString()方法搞得鬼？？？难怪在``BaseExecutor#close()``方法中的断点一直进不去，却修改了变量值。

**那为什么idea展示变量,调用toString()方法会导致此时查询所使用Executor被close呢?**
根据上面的提示，查看org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor源码，看看具体是什么逻辑

```java
private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
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
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator
              .translateExceptionIfPossible((PersistenceException) unwrapped);
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

从代码上看，这是jdk动态代理中的一个拦截器实现类，因为通过jdk动态代理，代理了Mybatis中的``SqlSession``接口,在idea中变量视图展示时被调用了toString()方法,导致被拦截。而invoke()方法中最后一定会在finally中关闭当前线程所关联的sqlSession,导致调用``BaseExecutor.close()``方法。为了验证这个想法，在SqlSessionInterceptor中对拦截到的toString()方法做了如下处理，如果是toString()方法不再向下继续执行,只要返回是哪些接口的代码类即可.

```java
private class SqlSessionInterceptor implements InvocationHandler {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

            if (args == null && "toString".equals(method.getName())) {
                return Arrays.toString(proxy.getClass().getInterfaces());
            }
           ... 其他代码省略
        }
}
```

恢复idea中的设置，再次调试，果然不会再出现Executor was closed.异常。
这看似mybatis-spring在实现SqlSessionInterceptor 时考虑不周全导致的一个bug，为了不泄露公司的框架代码还原这个bug，于是单独搭建了SpringBoot+Mybatis整合工程，并且写了一个类似逻辑的拦截器。代码如下:

```java
@Intercepts({
        @Signature(method = "query", type = Executor.class, args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(method = "query", type = Executor.class, args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),
        @Signature(method = "update", type = Executor.class, args = {MappedStatement.class, Object.class})
})
public class MybatisExecutorInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        Object proceed = invocation.proceed();
        Executor executor = (Executor) invocation.getTarget();
        Transaction transaction = executor.getTransaction();
        //其他逻辑略...
        return proceed;
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
}
```

再次在``SqlSessionInterceptor``中断点执行，经过几次debug，尝试还原这个bug时，程序尽然一路畅通完美通过,没有任何异常。
此刻我立刻想起了之前观察到的一段不合理代码，在文章开头的实例代码中``Executor ``被做为成员变量保存，**但是mybatis中``Interceptor``实现类是在程序启动时就被实例化的，并且是一个单实例对象。而在每次执行SQL时都会去创建一个新的``Executor``对象并且会经过``Interceptor``的``public Object plugin(Object target)``，用于判断是否需要对该Executor对象进行代理。** 而示例中重写的plugin方法，每次都对Executor重新赋值，**实际上这是线程不安全的**。由于在idea中debug时展示变量调用了toString()方法，同样会创建``SqlSession``、``Executor``经过plugin方法，导致Executor成员变量实际上是被替换的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200903200935668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyMDIyMzg=,size_16,color_FFFFFF,t_70#pic_center)
**解决方案**：直接通过``invocation.getTarget()``去获取被代理对象即可，而不是使用成员变量。

**为什么线上程序没有报``Executor was closed``问题???**

1. 因为线上不会像在idea中一样去调用toString() 方法
2. 代码中使用了缓存，当使用了Executor 获取到url后，下次请求过来就不会再使用Executor对象，也就不会出现异常。
3. 程序刚启动时并发量不够大，如果在程序刚起来时，立刻有足够的请求量，仍然会抛出异常，但是只要有一次结果被缓存，后续也就不会出现异常。

总结：
实际上还是MybatisExecutorInterceptor中将Executor做为成员变量，对Executor更改，出现线程不安全导致的异常。而idea中显示变量值调用toString()方法只是让异常发生的诱因。

原文链接：

https://blog.csdn.net/u013202238/article/details/108249483



