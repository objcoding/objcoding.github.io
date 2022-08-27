---
layout: post
title: "Go的new和make"
categories: Go
tags: new make 指针 引用类型
author: 张乘辉
---

* content
{:toc}
Go 语言中 new 和 make 都是内置函数，用于内存的分配，本文主要简述两者使用上的异同与特性。









## new

举个例子：

```go
func main() {
  var i *int
  *i = 1
  fmt.Println(*i)
}
```

上面的程序并不会打印1，而会抛 panic 异常，因为i是一个引用类型，需要给它分配内存空间，**通俗来说就是指针（内存地址）需要指向一片内存空间才有意义。**

为 i 分配内存：

```go
func main() {
  var i *int
  i = new(int)
  *i = 1
  fmt.Println(*i)
}
```

用 new 内置函数为 i 分配内存空间，并返回该内存空间的地址，即指针，new 函数格式如下：

```go
func new(Type) *Type
```

可知，new 为每个类型分配一片内存空间，初始化为 0 并返回该内存空间的地址。

new 的内存分配示意图：

![new](https://raw.githubusercontent.com/objcoding/md-picture/master/img/gonew.png)

其实要说明一点的就是，new 不常用，我们常常会通过结构体的字面量达到 new 的效果，而且这样写也比较优雅：

```go
man := &People{Name: "zhangchenghui", Age: 18, Sex: "男"}
```



## make

make 也是分配内存分配，但是仅限 chan、map、slice 的内存创建，并返回其**类型的引用**，这一点很重要， chan、map、slice 其本身已经是引用类型了，**所以make不需要再返回其指针，引用类型的本质就是指针**！例如：

```go
type i *int;
```

如上，i 就是一个自定义的引用类型，其类型是一个 int 类型的指针。

Make 内置函数格式：

```go
func make(t Type, size ...IntegerType) Type
```



make 的内存分配示意图：

![make](https://raw.githubusercontent.com/objcoding/md-picture/master/img/gomake.png)



