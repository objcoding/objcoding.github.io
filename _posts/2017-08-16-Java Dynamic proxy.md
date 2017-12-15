---
layout: post
title: "Java动态代理原理分析"
categories: Java
tags: proxy
author: zch
---

* content
{:toc}
代理模式是 Java 常用的一个设计模式，目的是为了生成一个代理的对象，去代理某个真实的对象，真正执行任务的是代理对象，调用方把这个代理对象当成真正的某个对象来执行。且代理对象还能在真实对象的基础上实现增强，Spring AOP 的底层原理就是 Java 动态代理模式。











## 动态代理类 Proxy 

Proxy 类中最重要的一个方法：

```java
Object proxyObject = Proxy.newProxyInstance(ClassLoader classLoader, Class[] interfaces, InvocationHandler h);
```

- ClassLoader：指定的类加载器，把指定代理类的 .class 文件加载到内存，形成 Class 对象；


- Class[] interfaces：指定要实现代理类的接口数组；


- InvocationHandler：一定要实现的一个接口，它只有一个方法（invoke）需要实现，实现方法就是动态代理的具体实现。

该方法动态创建实现了 interfaces 数组中所有指定接口的实现类对象。

方法源码：

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class>[] interfaces,
                                      InvocationHandler h)
  throws IllegalArgumentException
    {
		// 检查InvocationHandler不为空，如果为空则抛异常
        Objects.requireNonNull(h);

        final Class>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        // 利用指定类加载器与一组与指定接口相关代理类生成class对象
        Class<?> cl = getProxyClass0(loader, intfs);

		/*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }
			// 获取代理对象的构造方法  $Proxy0(InvocationHandler h)
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
			// 生成代理类的实例并把InvocationHandlerImpl的实例传给它的构造方法
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```



## InvocationHandler 接口

InvocationHandler 是代理实例的调用处理程序实现的接口。

**每个代理实例都具有一个关联的调用处理程序。对代理实例调用方法时，将对方法调用进行编码并将其指派到它的调用处理程序的 invoke 方法。**

```java
public Object invoke(Object proxy, Method method, Object[] args);
```

- Object proxy：当前对象，即代理对象，在调用谁的方法；
- Method method：当前被调用的方法（目标方法）；
- Object[] args：方法实参。

如下图所示：

![invocationHandler](https://raw.githubusercontent.com/zhangchenghuidev/zhangchenghuidev.github.io/master/images/proxy.png)



## 实战

定义一个接口：

```java
public interface User {
  public void eat();
}
```

定义一个 Man 类，实现 User 接口：

```java
public Man implements User {
  @Override
  public void eat() {
    System.out.println("吃饭");
  }
}
```

生成动态代理类： 

```java
User man = (Man) Proxy.newProxyInstance(User.class.getClassLoader(),
                                       new Class[]{User.class},
                                        (proxy, method, args) -> {
                                          	System.out.println("盛饭");
                                          	// 调用目标对象的目标方法
                                     		method.invoke(proxy, args);
                                          	System.out.println("喝水");
                                        });

// 这个方法会依次打印 “盛饭”，“吃饭”，“喝水”
man.eat();
```

