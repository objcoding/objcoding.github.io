---
layout: post
title: "Mybatis源码分析之Mapper实例化"
categories: Mybatis
tags: Mybatis Mapper SqlSession 动态代理
author: zch
---

* content
{:toc}
Mybatis 是一个「面向 sql」的持久层框架，它可实现动态拼装 sql，极其灵活，同时避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集，其插件机制允许在已映射语句执行过程中的某一点进行拦截调用等等，让我忍不住想要撸一撸它的源码。

我们都知道 Mapper 是一个接口，它的每个方式是我们与数据库交互的入口，每个 Mapper 都有与之相对应的一个 XML 文件，我们可以在 XML 里面自由快活地写 SQL，当然我们也可以用注解的形式写在接口方法上，但终究还是没 XML 灵活，那么问题来了，Mybatis 是如何实例化 Mapper 的呢？下面我带你揭开这个神秘的面纱。









## Mapper 实例化过程

首先我们来看看用 Mybatis 执行 sql 的两种方法

- 直接操作 SqlSession 方法

```Java
public User findUserById(Integer userId) {
  SqlSession sqlSession = MyBatisSqlSessionFactory.getSqlSession();
  try {
    // namespace + statementId
    return sqlSession.selectOne("com.objcoding.mybatis.UserMapper.findUserById", userId);
  } finally {
    sqlSession.close();
  }
}
```

- 通过 Mapper 接口

```java
public User findUserById(Integer userId) {
  SqlSession sqlSession = MyBatisSqlSessionFactory.getSqlSession();
  try {
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    return userMapper.findUserById(userId);
  } finally {
    sqlSession.close();
  }
}
```

```java
public class UserMapper {
  User findUserById(@Param("userId") String userId);
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.objcoding.mybatis.UserMapper">
  <select id="findUserById" resultType="com.objcoding.mybatis.User">
    SELECT * FROM user WHERE user_id=#{userId}
  </select>
</mapper>
```

很明显，第二种方法可以大大降低了手工写 namespace 出现错误的概率，且用 Mapper 可以直接操作方法来实现数据链接，看起来优雅很多。

那么 Mapper 是如何示例化的，它是通过 Java 动态代理生成的一个代理类，并与 sqlSession 关联一起，看如下图：

![Mapper 代理类](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/mybatis.ng)

看得出来，此时的 Mapper 是 Spring Bean 容器里面的一个 Bean，它是一个代理类，那么这个代理类的生成过程是怎样的呢？下面带你一起看看 mybatis 源码。



如果是 Spring 体系的项目，一般都创建一个 SqlSessionFactory 实例并注入 Bean 容器中：

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

继续看 bean.getObject()：

```java
@Override
public SqlSessionFactory getObject() throws Exception {
  if (this.sqlSessionFactory == null) {
    afterPropertiesSet();
  }

  return this.sqlSessionFactory;
}
```

```java
@Override
public void afterPropertiesSet() throws Exception {
  // 此处省略部分代码
  this.sqlSessionFactory = buildSqlSessionFactory();
}
```

看到了创建 sqlSessionFactory 的方法：

```java
protected SqlSessionFactory buildSqlSessionFactory() throws Exception {
  Configuration configuration;
  // 此处省略部分代码
  SqlSessionFactory sqlSessionFactory =this.sqlSessionFactoryBuilder.build(configuration);
  // 此处省略部分代码
  if (!isEmpty(this.mapperLocations)) {
	// 此处省略部分代码
    try {
      XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(), configuration, mapperLocation.toString(), configuration.getSqlFragments());
      xmlMapperBuilder.parse();
    } catch (Exception e) {
      throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
    } finally {
      ErrorContext.instance().reset();
    }
    // 此处省略部分代码
return sqlSessionFactory;
}
```

看到这里 xmlMapperBuilder 





### 生成 Mapper 代理工厂

 



### 保存 MappedStatement 映射器节点







### MapperMethod 委托 SqlSession 去执行 sql









## mybatis-spring 注入 Mapper bean



