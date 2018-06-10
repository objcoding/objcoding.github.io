---
layout: post
title: "Mybatis-spring源码分析之注册Mapper Bean"
categories: Mybatis
tags: Mybatis Spring IOC 源码分析
author: zch
---

* content
{:toc}
上一篇讲到了 Mapper 如何注册到 Configuration 类中与 MapperProxyFactory 绑定的过程，我们平时的使用场景一般都配合着 Spring，使用 Spring 意味着需要注册 Bean，也就是说需要将 Mapper 的代理实现类注册到 Spring 容器中，成为 Spring 容器中的一个 Bean。 











## Mybatis 单独使用方式

如果是单独使用 Mybatis，需要手动创建 Mapper 代理实现类：

```java
// 以下是半伪代码

// 创建 sqlSessionFactory 工厂类
SqlSessionFactory sqlSessionFactory = SqlSessionFactoryBuilder.build(configuration);

// 创建一个 sqlSession 客户端连接类
SqlSession sqlSession = sqlSessionFactory.openSession();

try {
  // 获取 Mapper 代理实现类
  XXXMapper mapper = sqlSession.getMapper(XXXMapper.class);
  // 操作数据库
  mapper.insert(xxx);
  sqlSession.commit();
} finally {
  sqlSession.close();
}

```

很显然这样使用 Mybatis 是代码略显臃肿，且不美观，我们接下来看看 Spring 是如何优雅地用使用 Mybatis。



## Spring 注册 Mapper Bean

[mybatis-spring](https://github.com/mybatis/spring) 提供了注册 Mapper Bean 的功能，这里涉及 Bean 的注册与加载过程，因此有很多接口需要在这里详细解析一下：

### SqlSessionFactoryBean

在上一篇的源码分析中已经有提及到 SqlSessionFactory 的实例了，创建 SqlSessionFactory 实例包括了 Mapper 的注册和绑定过程：

```java
@Bean
public SqlSessionFactory sqlSessionFactory() throws Exception {
  PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
  SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
  
  // 此处省略部分代码
  
  bean.setMapperLocations(resolver.getResources("classpath*:com/**/*Mapper.xml"));//
  return bean.getObject();
}
```

这里我们主动创建 SqlSessionFactory 的实例并注册到 Bean 容器中。

SqlSessionFactory 是 Mybatis 应用的核心类，它是创建 SqlSesison 的工厂类，而 SqlSesison 是我们用 Mybatis 与数据库会话的 顶层 API 类，所有与数据库会话都需要创建 SqlSesison。

SqlSessionFactoryBean 是 SqlSessionFactory 的具体实现，他实现了 FactoryBean 和 InitializingBean 接口：

- **FactoryBean：BeanFactory 定义了 Ioc 容器的最基本形式，并提供了 Ioc 容器应遵守的的最基本的接口，也就是 Spring Ioc 所遵守的最底层和最基本的编程规范，一旦某个 Bean 实现了该接口，Spring 的getBean方法其实是调用了 Bean 的 getObject() 方法，即是获取 Bean 实例是通过 getObject() 获取的，这规范很重要，后面会重点围绕着点来讲。**

```java
public interface FactoryBean<T> {

  // 返回一个由 FactoryBean 创建的 Bean 实例
  T getObject() throws Exception;

  // 返回一个由 FactoryBean 创建的 Bean 实例的类型
  Class<?> getObjectType();

  // FactoryBean 创建的 Bean 实例是否是单例
  boolean isSingleton();

}
```



- InitializingBean：某个 Bean 若实现了 该接口，那么 Spring 实例该 Bean 时首先会调用 afterPropertiesSet() 方法对 Bean 进行初始化动作。

SqlSessionFactoryBean 的初始化：

```java
@Override
public void afterPropertiesSet() throws Exception {
  // 此处省略部分代码
  this.sqlSessionFactory = buildSqlSessionFactory();
}
```

获取 SqlSessionFactoryBean 实例：

```java
@Override
public SqlSessionFactory getObject() throws Exception {
  if (this.sqlSessionFactory == null) {
    afterPropertiesSet();
  }

  return this.sqlSessionFactory;
}
```





### MapperFactoryBean

Mapper的代理实现类都是有 MapperFactoryBean 来实现了的，它同样实现了 FactoryBean 和 InitializingBean 两个接口，它还继承了 SqlSessionDaoSupport， 该类是 SqlSession 生成的支持帮助类，其中有两个关键的方法：

```java
public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
  if (!this.externalSqlSession) {
    this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
  }
}

public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
  this.sqlSession = sqlSessionTemplate;
  this.externalSqlSession = true;
}
```

在 Bean 加载的过程中，Spring 的 PropertyAccessor 实现类会自动设置 Bean 的属性值：

```java
void setPropertyValue(String propertyName, @Nullable Object value) throws BeansException;  
```

**不难看出，SqlSessionFactoryBean 被加载时，需要 SqlSessionFactory 或者 SqlSessionTemplate 的实现类**，所以我们在 Mapper Bean 加载之前，需要需要手动生成其中的一个，上一篇我们特别介绍了主动生成了 SqlSessionFactory 实现类的过程分析。以下是 MybatisAutoConfiguration 自动话配置类 SqlSessionTemplate Bean 的创建方法：

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
  ExecutorType executorType = this.properties.getExecutorType();
  if (executorType != null) {
    return new SqlSessionTemplate(sqlSessionFactory, executorType);
  } else {
    return new SqlSessionTemplate(sqlSessionFactory);
  }
}
```

从 externalSqlSession 字段可知，如果 Spring 容器中 SqlSessionFactory 和 SqlSessionTemplate同时存在，那么  SqlSessionDaoSupport 的这两个属性只会被设置一次。



如果是用标签方式，需要这样配置来注入属性值（不推荐标签配置 Bean 的方式，因为与时代严重脱轨了）：

```java
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
    <property name="mapperInterface" value="com.tqd.dao.UserMapper"></property>
    <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

获取 MapperFactoryBean 实例：

```java
@Override
public T getObject() throws Exception {
  return getSqlSession().getMapper(this.mapperInterface);
}
```

该方法里面这段代码，就是我们单独使用 Mybatis 时的一个代码调用方式，这里 Spring 巧妙地在这里进行了封装，因为 MapperFactoryBean 实现了 FactoryBean 接口，Spring 加载 Bean时，实际上是调用 FactoryBean 的getObject() 方法，到这里你似乎有点豁然开朗的感觉了是不是？



### SqlSessionTemplate

SqlSessionTemplate 是 SqlSession 的一个实现类，也是 Mybatis-spring 的核心类，Spring 整合 Mybatis 的最终目的无非就是创建 SqlSessionTemplate 与数据进行会话动作，来看看 SqlSessionTemplate 的构造方法：

```java
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

我们创建 SqlSessionTemplate 实例最终是调用该方法来实现的，这里将 sqlSessionFactory 给赋值了，而且还通过反射生成了一个 SqlSession 代理类，







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





### MapperScan 注解扫描入口





## 后记

用到了很多 spring 的类：FactoryBean、DaoSupport、InitializingBean、ImportBeanDefinitionRegistrar、BeanDefinitionRegistryPostProcessor、ClassPathBeanDefinitionScanner、BeanDefinitionRegistry、BeanDefinitionHolder、DefaultListableBeanFactory（beanFactory）