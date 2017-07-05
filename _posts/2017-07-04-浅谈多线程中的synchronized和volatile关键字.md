---
layout: post
title: "浅谈多线程中的synchronized和volatile关键字"
categories: Java
tags: 多线程 synchronized volatile
author: zch
---

* content
{:toc}
>  synchronized主要为线程提供同步功能，而volatile主要是设置参数的可见性。





## synchronized

在并发编程过程中，很多会涉及到线程安全问题，“非线程安全”指的是多个线程访问同一个对象中的实例变量进行并发访问时发生的“脏读”，而“线程安全”指的是多个线程同时访问的变量为私有变量，并不会共享，不会产生脏读现象，也就是说如果变量在方法内，就不会产生“非线程安全”的问题了。

为了解决示例变量可能出现的“非线程安全”的问题，我们需多线程关键字synchronized，



## volatile



