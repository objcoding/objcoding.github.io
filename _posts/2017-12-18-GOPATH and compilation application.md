---
layout: post
title: "GOPATH与编译应用"
categories: Go
tags: GOPATH
author: zch
---

* content
{:toc}
我已经被Go语言的大道至简的设计深深地吸引了，它自带的命令诸如go run、go build、go install等，就可以编译运行Go应用了，这在java中，我们还需要依赖maven的编译工具，Go的这些命令就相当于一个maven了，甚至比maven简单多了，而且还是原生支持，这篇文章主要是说一下Go的工作目录与编译的规则，初步体验一下Go的大道至简的魅力。











## GOPATH

GOPATH是go命令依赖的一个路径，也是Go项目放置的地方，在类unix系统下设置GOPATH：

```bash
export GOPATH=/Users/zhangchenghui/.go
```

查看Go环境变量：

```bash
go env
```

![gopath](https://raw.githubusercontent.com/zhangchenghuidev/zhangchenghuidev.github.io/master/images/go.png)



$GOPATH目录有三个约定俗成的目录，一定要彻底理解：

- src：存放源代码，也是 Go 项目源代码的存放地址；
- pkg：编译后的生成的包，也就是Go的.a文件，这个后缀名的文件代表的是Go的一个包；
- bin：编译后生成的可执行文件（只有导入 package main 包的文件编译后直接是可执行文件 ）。



![gopath](https://raw.githubusercontent.com/zhangchenghuidev/zhangchenghuidev.github.io/master/images/go2.png)



## 编译应用

在GOPATH的src创建一个项目：

```bash
cd $GOPATH/src
mkdir mypakage
cd mypakage
```

新建 test.go：

```go
package mypakage

import "fmt"
  
func test() {
     fmt.Println("test~~~~~")
}
```

在该项目目录中执行 go install 或者在任意目录下执行 go install mypakage，请注意该文件的包 package mypakage，意味着编译后会在pkg目录生成一个包。

![gopath](https://raw.githubusercontent.com/zhangchenghuidev/zhangchenghuidev.github.io/master/images/go3.png)

接下来我们就可以引用这个包里面的方法啦，这和maven 的 mvn clean install 一个道理。

在 src 目录中新建一个应用：

```bash
cd $GOPATH/src
mkdir myapp
cd myapp
```

新建main.go：

```bash
vim main.go
```

```Go
package main

import (
	"fmt"
  	"mypakage"
)

func main() {
  mypakage.test()
  fmt.Println("hello, go")
}
```







