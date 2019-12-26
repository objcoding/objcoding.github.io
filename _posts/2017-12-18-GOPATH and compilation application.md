---
layout: post
title: "Go的环境配置与应用编译"
categories: Go
tags: GOPATH
author: zch
---

* content
{:toc}
我已经被 Go 语言的大道至简的设计深深地吸引了，它自带的命令诸如 go run、go build、go install 等，就可以编译运行 Go 应用了，这在 Java 中，我们还需要依赖 maven 的编译工具，Go 的这些命令就相当于一个 maven了，甚至比 maven 简单多了，而且还是原生支持，这篇文章主要是说一下 Go 的工作目录与编译的规则，初步体验一下 Go 的大道至简的魅力。











## GOPATH

GOPATH 是 Go 命令依赖的一个路径，也是 Go 项目放置的地方，在类 unix 系统下设置 GOPATH：

```bash
export GOPATH=/Users/zhangchenghui/.go
```

查看 Go 环境变量：

```bash
go env
```

![gopath](https://gitee.com/objcoding/md-picture/raw/master/img/go.png)



$GOPATH 目录有三个约定俗成的目录，一定要彻底理解：

- src：存放源代码，也是 Go 项目源代码的存放地址；
- pkg：编译后的生成的包，也就是 Go 的 .a 文件，这个后缀名的文件代表的是 Go  的一个包；
- bin：编译后生成的可执行文件（**只有导入 package main 包的文件编译后直接是可执行文件** ）。



![gopath](https://gitee.com/objcoding/md-picture/raw/master/img/go2.png)



## 应用编译

在 GOPATH 的 src 创建一个项目：

```bash
cd $GOPATH/src
mkdir mypakage
cd mypakage
```

新建 test.go：

```go
// $GOPATH/src/mypakage/test.go
package mypakage

import "fmt"

func Test() {
     fmt.Println("test~~~~~")
}
```

Go 语言有个约定俗成的做法就是函数首字母大写相当于 Java 的 public 方法，小写相当于 Java 的 private 方法。

在该项目目录中执行 go install 或者在任意目录下执行 go install mypakage，请注意该文件的包 package mypakage，意味着编译后会在 pkg 目录生成一个包。

![gopath](https://gitee.com/objcoding/md-picture/raw/master/img/go3.png)

接下来我们就可以引用这个包里面的方法啦，这和 maven 的 mvn clean install 一个道理。

在 src 目录中新建一个应用：

```bash
cd $GOPATH/src
mkdir myapp
cd myapp
```

新建 main.go：

```go
// $GOPATH/src/myapp/main.go
package main

import (
	"fmt"
  	"mypakage"
)

func main() {
  mypakage.Test()
  fmt.Println("hello, go")
}
```



接下来就是要编译这个应用了，进入该应用目录，执行 go install，**由于该应用直接导入的是pakage main 包，它是Go语言中唯一一个可以编译后直接生成可执行文件的包**，因此会在 $GOPATH/bin 下生成一个可执行文件  myapp：

![gopath](https://gitee.com/objcoding/md-picture/raw/master/img/go4.png)

在任意路径下，执行：

```
myapp
```

输出如下内容：

![gopath](https://gitee.com/objcoding/md-picture/raw/master/img/go5.png)



其实在该应用目录下也可执行 go build 命令进行编译，会在当前目录下生成可执行文件，而不会安装在 bin 目录下。

![gopath](https://gitee.com/objcoding/md-picture/raw/master/img/go6.png)





## 拉取远程包

Go 语言要如何获取远程包呢？在 Java 开发中，我们我可以通过 maven 自动地从 maven 中央仓库中下载依赖到.m2本底仓库中，但是在 Go 开发中，我们只需要使用 go get 指令就可以从远程拉取依赖包了：

```bash
go get github.com/astaxie/beego
```

这条命令就会将源码下载到 src 目录中，并将源码编译后安装到 pkg 目录中：

![gopath](https://gitee.com/objcoding/md-picture/raw/master/img/go7.png)

因此，go get 相当于 git clone 源码下来，再执行 go install。