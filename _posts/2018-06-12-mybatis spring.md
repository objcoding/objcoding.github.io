---
layout: post
title: "Mybatis-spring源码分析之注册Mapper Bean"
categories: Mybatis Spring
tags: Mybatis Spring IOC 反射 源码分析
author: 张乘辉
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

从 externalSqlSession 字段可知，如果 Spring 容器中 SqlSessionFactory 和 SqlSessionTemplate 同时存在，那么  SqlSessionDaoSupport 的这两个属性只会被设置一次。



如果是用标签方式，需要这样配置来注入属性值（不推荐标签配置 Bean 的方式，因为与时代严重脱轨了）：

```java
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
    <property name="mapperInterface" value="com.objcoding.dao.UserMapper"></property>
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

该方法里面这段代码，就是我们单独使用 Mybatis 时的一个代码调用方式，这里 Spring 巧妙地在这里进行了封装，因为 MapperFactoryBean 实现了 FactoryBean 接口，Spring 加载 Bean 时，实际上是调用 FactoryBean 的getObject() 方法，到这里你似乎有点豁然开朗的感觉了是不是？



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

我们创建 SqlSessionTemplate 实例最终是调用该方法来实现的，sqlSessionFactoryBean 实例赋值给 SqlSessionTemplate 的 sqlSessionFactory 属性，而且还通过反射生成了一个 SqlSession 代理类，该代理类即是与数据会话的关键：

```java
@Override
public <T> T selectOne(String statement, Object parameter) {
  return this.sqlSessionProxy.<T> selectOne(statement, parameter);
}
```

继续往下看：

```java
private class SqlSessionInterceptor implements InvocationHandler {
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

    // 创建 DefaultSqlSession 实现类（它不是线程安全的）
    SqlSession sqlSession = getSqlSession(
      SqlSessionTemplate.this.sqlSessionFactory,
      SqlSessionTemplate.this.executorType,
      SqlSessionTemplate.this.exceptionTranslator);
    try {
      // 通过反射调用 sqlSession 具体方法
      Object result = method.invoke(sqlSession, args);
      if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
        // force commit even on non-dirty sessions because some databases require
        // a commit/rollback before calling close()
        // 提交执行
        sqlSession.commit(true);
      }
      // 返回执行结果
      return result;
    } catch (Throwable t) {
      Throwable unwrapped = unwrapThrowable(t);
      //然后判断一下当前的sqlSession是否被Spring托管 如果未被Spring托管则自动commit
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
      //关闭sqlSession
      if (sqlSession != null) {
        closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
      }
    }
  }
}
```

getSqlSession 创建一个 SqlSession 实现类，这里是一个默认的实现类 DefaultSqlSession，从该反射具体执行类看出，我们调用代理类执行操作时，已经自动给执行了 commit、close 等操作了，当前的 sqlSession 是否被 Spring 托管，还有事务回滚功能，大大节省了代码量有木有！





### MapperScan 注解扫描入口

接下来就是重头戏了，我们此时需要将 Mapper 接口类扫描并注册生成对应的 BeanDefinition 并添加到 BeanDefinitionRegistry 中。

BeanDefinition：**它是 Spring 中用于包装 Bean 的数据结构，一个 BeanDefinition 描述了一个 bean 的实例，包括它的类名，具体的 class 对象等属性值。**

- MapperScan 注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
public @interface MapperScan {

  String[] value() default {};

  // 省略部分代码

  Class<? extends MapperFactoryBean> factoryBean() default MapperFactoryBean.class;

}
```

在这个注解里面包含了 @Import 注解，它的作用是导入资源，如果导入的是一个普通类，spring 还会将其注册成一个普通的 Bean，通过 @Import 注入了 Mapper 扫描注册类，通过该扫描类扫描 Mapper 目录，并将 Mapper 注册成一个 Bean。

- MapperScannerRegistrar

```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {

  private ResourceLoader resourceLoader;

  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    // 获取当前注解信息
    AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    // 创建 Mapper 扫描类，该类会自动扫描指定目录并将 Mapper 封装成 一个个 BeanDefinition 并添加到 BeanDefinitionRegistry 中
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    // 省略部分代码
    
    // 执行过滤
    scanner.registerFilters();
    // 开始扫描
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }

  @Override
  public void setResourceLoader(ResourceLoader resourceLoader) {
    this.resourceLoader = resourceLoader;
  }

}
```

**MapperScannerRegistrar 类实现了 ImportBeanDefinitionRegistrar ，该 interface 是 Spring 动态注册 Bean 的方法，所有实现了该接口的类都会被 ConfigurationClassPostProcessor 处理。**

- ClassPathMapeerScanner

```java
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
  
  // 省略部分代码
  
  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // 调用父类方法对指定路径进行扫描并注册 BeanDefinition
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }

  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
     
      // 省略部分代码
      
      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      //将其bean Class 类型设置为 mapperFactoryBean，放入 BeanDefinitions definition.
 definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName());
      definition.setBeanClass(this.mapperFactoryBean.getClass());

      // 省略部分代码
      
    }
  }

  // 省略部分代码

}
```

**ClassPathMapeerScanner 继承了 ClassPathBeanDefinitionScanner 类，ClassPathBeanDefinitionScanner 会扫描base-package下的所有spring定义的注解标识类，也可以对扫描的机制进行配置，设置一些 Filter，只有满足Filter的类才能被注册为Bean。**

**这里实现并覆盖了 doScan 方法的，并将其 BeanDefinition 中的 beanClass 设置为 MapperFactoryBean，这一步特别关键啊，因为 MapperFactoryBean 实现了 BeanFactory 接口，spring 加载这类型的 Bean 会调用其 getObejct() 方法，上面也提到过 MapperFactoryBean 的 getObejct() 实则是调用 SqlSession 的 getMapper() 方法生成 Mapper 代理类，而该代理类底层操作的还是 SqlSession，绕了一大圈，终于搞明白 Spring 的「用心良苦」了吧。**



在分析源码时习惯胡乱地画一下：

![mybatis-spring](https://gitee.com/objcoding/md-picture/raw/master/img/mybatis-spring.jpeg)

