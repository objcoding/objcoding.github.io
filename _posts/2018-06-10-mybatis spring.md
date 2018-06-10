---
layout: post
title: "Mybatis-spring源码分析之注入Mapper Bean"
categories: Mybatis
tags: Mybatis Spring IOC 源码分析
author: zch
---

* content
{:toc}
上一篇讲到了 Mapper 如何注册到 Configuration 类中与 MapperFactoryBean 绑定的过程，我们平时的使用场景一般都配合着 Spring，使用 Spring 意味着需要注册 Bean，也就是说需要将 Mapper 的代理实现类注册到 Spring 容器中，成为 Spring 容器中的一个 Bean。 











## Mybatis 单独使用

如果是单独使用 Mybatis，需要手动创建 Mapper 代理实现类：

```java
// 以下是伪代码

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



## Spring 注入 Mapper Bean



### SqlSessionFactory

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

SqlSession 





### MapperFactoryBean





### MapperScan 注解





## 后记

用到了很多 spring 的类：FactoryBean、DaoSupport、InitializingBean、ImportBeanDefinitionRegistrar、BeanDefinitionRegistryPostProcessor、ClassPathBeanDefinitionScanner、BeanDefinitionRegistry、BeanDefinitionHolder、DefaultListableBeanFactory（beanFactory）