---
layout: post
title: "ApplicationContextAware的作用"
categories: Java
tags: Spring
author: zch
---

* content
{:toc}
## 1. ApplicationContextAware接口

ApplicationContextAware的作用是可以方便获取Spring容器ApplicationContext，从而可以获取容器内的Bean。

```java
public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext var1) throws BeansException;
}
```

ApplicationContextAware接口只有一个方法，如果实现了这个方法，那么Spring创建这个实现类的时候就会自动执行这个方法，把ApplicationContext注入到这个类中，也就是说，spring 在启动的时候就需要实例化这个 class（如果是懒加载就是你需要用到的时候实例化），在实例化这个 class 的时候，发现它包含这个 ApplicationContextAware 接口的话，sping 就会调用这个对象的 setApplicationContext 方法，把 applicationContext Set 进去了。

## 2. 配置Spring的XML配置

