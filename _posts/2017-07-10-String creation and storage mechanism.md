---
layout: post
title: "字符串创建与存储机制"
categories: Java
tags: String
author: zch
---

* content
{:toc}
在Java语言中，字符串是一种不可变类，即它是被final修饰过的类，因此也不可继承。







```java
String s1 = new String("abc");// 在堆内存中创建新的对象
String s2 = new String("abc");// 在堆内存中又创建了一个新的对象
```

上面两个式子，存量两个对象引用s1和s2，和内容相同的字符串对象“abc”，其中s1与s2是两个不同的对象，因为new总是生成新的对象。

```java
String s3 = "abc";// 在常量区创建了一个"abc"字符串对象
String s4 = "abc";// s4引用了在常量区中"abc"字符串对象，这里并不会产生新的对象
```

在jvm内存中，存在着一个字符串常量池，保存着很多String对象，且共享，s3与s4同样引用的是常量池中的”abc“对象，每次创建字符串时，jvm都会先检查 一下常量池是否又这个字符串对象，且创建好以后都不可修改；如果没有则创建，如果有则直接获取其引用。

![字符串存储机制](https://gitee.com/objcoding/md-picture/raw/master/img/string.png)

