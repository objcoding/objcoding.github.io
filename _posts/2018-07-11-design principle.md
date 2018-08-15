---
layout: post
title: "Java设计模式——六大设计原则"
categories: Design
tags: 设计原则
author: zch
---

* content
{:toc}




















### 单一职责原则（Single Responsibility Principle - SRP）

> 原文：There should never be more than one reason for a class to change.
>
> 译文：永远不应该有多于一个原因来改变某个类。

理解：对于一个类而言，应该仅有一个引起它变化的原因。说白了就是，不同的类具备不同的职责，各施其责。这就好比一个团队，大家分工协作，互不影响，各做各的事情。

应用：当我们做系统设计时，如果发现有一个类拥有了两种的职责，那就问自己一个问题：可以将这个类分成两个类吗？如果真的有必要，那就分吧。千万不要让一个类干的事情太多！

### 开放封闭原则（Open Closed Principle - OCP）

> 原文：Software entities like classes, modules and functions should be open for extension but closed for modifications.
>
> 译文：软件实体，如：类、模块与函数，对于扩展应该是开放的，但对于修改应该是封闭的。

理解：简言之，对扩展开放，对修改封闭。换句话说，可以去扩展类，但不要去修改类。

应用：当需求有改动，要修改代码了，此时您要做的是，尽量用继承或组合的方式来扩展类的功能，而不是直接修改类的代码。当然，如果能够确保对整体架构不会产生任何影响，那么也没必要搞得那么复杂了，直接改这个类吧。

### 里氏替换原则（Liskov Substitution Principle - LSP）

> 原文：Functions that use pointers or references to base classes must be able to use objects of derived classes without knowing it.
>
> 译文：使用基类的指针或引用的函数，必须是在不知情的情况下，能够使用派生类的对象。

理解：父类能够替换子类，但子类不一定能替换父类。也就是说，在代码中可以将父类全部替换为子类，程序不会报错，也不会在运行时出现任何异常，但反过来却不一定成立。

应用：在继承类时，务必重写（Override）父类中所有的方法，尤其需要注意父类的 protected 方法（它们往往是让您重写的），子类尽量不要暴露自己的 public 方法供外界调用。

### 最少知识原则（Least Knowledge Principle - LKP）

> 原文：Only talk to you immediate friends.
>
> 译文：只与你最直接的朋友交流。

理解：尽量减少对象之间的交互，从而减小类之间的耦合。简言之，一定要做到：低耦合，高内聚。

应用：在做系统设计时，不要让一个类依赖于太多的其他类，需尽量减小依赖关系，否则，您死都不知道自己怎么死的。

### 接口隔离原则（Interface Segregation Principle - ISP）

> 原文：The dependency of one class to another one should depend on the smallest possible interface.
>
> 译文：一个类与另一个类之间的依赖性，应该依赖于尽可能小的接口。

理解：不要对外暴露没有实际意义的接口。也就是说，接口是给别人调用的，那就不要去为难别人了，尽可能保证接口的实用性吧。她好，我也好。

应用：当需要对外暴露接口时，需要再三斟酌，如果真的没有必要对外提供的，就删了吧。一旦您提供了，就意味着，您将来要多做一件事情，何苦要给自己找事做呢。

### 依赖倒置原则（Dependence Inversion Principle - DIP）

> 原文：High level modules should not depends upon low level modules. Both should depend upon abstractions. Abstractions should not depend upon details. Details should depend upon abstractions.
>
> 译文：高层模块不应该依赖于低层模块，它们应该依赖于抽象。抽象不应该依赖于细节，细节应该依赖于抽象。

理解：应该面向接口编程，不应该面向实现类编程。面向实现类编程，相当于就是论事，那是正向依赖（正常人思维）；面向接口编程，相当于通过事物表象来看本质，那是反向依赖，即依赖倒置（程序员思维）。

应用：并不是说，所有的类都要有一个对应的接口，而是说，如果有接口，那就尽量使用接口来编程吧。

原文链接：https://mp.weixin.qq.com/s/Q2lNFHfpjveEMIR9Mmjrcg