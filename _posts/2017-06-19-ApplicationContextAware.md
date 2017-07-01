---
layout: post
title: "ApplicationContextAware的作用"
categories: Java
tags: Spring
author: zch
---

* content
{:toc}
ApplicationContextAware的作用是可以方便获取Spring容器ApplicationContext，从而可以获取容器内的Bean。

```java
public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext var1) throws BeansException;
}
```

ApplicationContextAware接口只有一个方法，如果实现了这个方法，那么Spring创建这个实现类的时候就会自动执行这个方法，把ApplicationContext注入到这个类中，也就是说，spring 在启动的时候就需要实例化这个 class（如果是懒加载就是你需要用到的时候实例化），在实例化这个 class 的时候，发现它包含这个 ApplicationContextAware 接口的话，sping 就会调用这个对象的 setApplicationContext 方法，把 applicationContext Set 进去了。





## 配置Spring的XML配置

我们需要在web项目启动的时候就启动Spring容器，所以需要在web.xml中配置Spring监听器：

```xml
<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

只会读取默认路径下Bean的application.xml配置文件的,如果需要读取特定路径下的配置文件,需要在web.xml中自定义配置：

```xml
<context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>classpath:config/spring/spring.xml</param-value>
</context-param>
```

## 创建AppUtil类

创建AppUtil并实现ApplicationContextAware接口：

```java
public class AppUtil implements ApplicationContextAware {

  private static ApplicationContext applicationContext = null;

  @Override
  public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    if (SpringBootUtil.applicationContext == null) {
      SpringBootUtil.applicationContext = applicationContext;
    }
  }
  
  //获取applicationContext
  public static ApplicationContext getApplicationContext() {
    return applicationContext;
  }

  //通过name获取 Bean.
  public static Object getBean(String name) {
    return getApplicationContext().getBean(name);
  }

  //通过class获取Bean.
  public static <T> T getBean(Class<T> clazz) {
    return getApplicationContext().getBean(clazz);
  }
  
  /**
     * 同步方法注册bean到ApplicationContext中
     *
     * @param beanName
     * @param clazz
     * @param original bean的属性值
     */
  public static synchronized void setBean(String beanName, Class<?> clazz, Map<String,Object> original) {
    checkApplicationContext();
    DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) applicationContext.getAutowireCapableBeanFactory();
    if(beanFactory.containsBean(beanName)){
      return;
    }
    //BeanDefinition beanDefinition = new RootBeanDefinition(clazz);
    GenericBeanDefinition definition = new GenericBeanDefinition();
    //类class
    definition.setBeanClass(clazz);
    //属性赋值
    definition.setPropertyValues(new MutablePropertyValues(original));
    //注册到spring上下文
    beanFactory.registerBeanDefinition(beanName, definition);
  }
  
}
```

最后我们需要把AppUtil作为普通的Bean注入到Spring容器中，需要在application.xml中配置：

```xml
<bean id="appUtil" class="com.util.AppUtil"/>
```

