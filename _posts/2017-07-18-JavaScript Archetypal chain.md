---
layout: post
title: "JavaScript原型链"
categories: JavaScript
tags: prototype
author: zch
---

* content
{:toc}
原型链可以用作js中继承的主要方法，其思路是把当前原型对象等于需要继承的对象，从而使当前原型的指针拥有需要继承的对象的原型的引用，若这样继续继承下去，层层递进，就可构成原型链。











```javascript
function Super() {
  this.property = false;
}
Super.prototype.getProperty = function(){
  return this.property;
};
function Sub() {
  this.subProperty = true;
  this.name = "zhangsan";
}
Sub.prototype = new Super();// 原型对象等于Super，使之拥有Super中所有属性和方法(包括Super的prototpe)
Sub.prototype.getSubProperty = function () {// 在原型基础上添加新的方法
  return this.subProperty;
};
var test = new Sub();// 创建Sub对象
test.getName = function () {// 在当前象中添加新方法，只有当前对象拥有
  return this.name;
};
var b = test.getProperty();// 通过原型链找到getProperty()方法, 该方法位于Object对象中
var n = test.getName();// 该方法位于Sub对象中
```



![prototype](https://gitee.com/objcoding/md-picture/raw/master/img/prototype.png)

![prototype](https://gitee.com/objcoding/md-picture/raw/master/img/prototype2.png)