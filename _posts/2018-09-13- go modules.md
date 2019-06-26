---
layout: post
title: "Go Modules踩坑总结"
categories: Go
tags: modules
author: zch
---

* content
{:toc}
在 Java 的项目中，有 Maven 和 Gradle 这些很好用的依赖版本管理工具，简直不要太方便了，但是在之前的 Golang 版本中，官方并没有提供版本管理工具，我们以前用 go get 获取依赖其实是有潜在危险的，因为我们不确定最新版依赖是否会破坏掉我们项目对依赖包的使用方式，即当前项目可能会出现不兼容最新依赖包的问题。之后官方出了一个 vendor 机制，将项目依赖的包都放在该目录中，但这也并没有很好地管理依赖的版本。之后官方出了一个准官方版本管理工具 go dep，这也算是 go modules 的前身了吧。随着 Go1.11 的发布，Golang 给我们带来了 module 全新特性，这是 Golang 新的一套依赖管理系统。现在 Go1.12 已经发布了，go modules 进一步稳定，但官方还是没有将其设为默认机制，所以踩坑之路是必须的，本篇文章除了详细说明 go modules 的特性以及使用之外，还总结了我在这个过程中遇到的一些“坑”。











## 创建 module

- 创建项目

在默认情况下，$GOPATH 默认情况下是不支持 go mudules 的，我们需要在项目目录下手动执行以下命令：

```bash
$ export GO111MODULE=on
```

**这也表明了 go 要利用 modules 机制消灭 $GOPATH 的决心啊！**

为了配合 go modules 机制，我们 $GOPATH 以外的目录创建一个 testmod 的包：

```bash
$ mkdir testmod
$ cd testmod

$ echo 'package testmod
		import "fmt"
	func SayHello(name string) string {
   return fmt.Sprintf("Hello, %s", name)
}' >> testmod.go
```

- 初始化 module：

```
$ go mod init github.com/objcoding/testmod
go: creating new go.mod: module github.com/objcoding/testmod
```

以上命令会在项目中创建一个 go.mod 文件，初始化内容如下：

```
module github.com/objcoding/testmod
```

这时，我们的项目已经成为了一个 module 了。

- 推送到 github 仓库

```bash
$ git init
$ git add *
$ git commit -am "First commit"
$ git push -u origin master
```



在这里我也着重说下关于项目依赖包引用地址的问题，这个问题虽小，但也确实很困扰人，所以必须得说一下：

go mudules 出现之前，在一个项目中有很多个包，在项目内，有些包需要依赖项目内其它包，假设项目有个包，相对于 gopath 的地址是 objcoding/mypackage，在项目内其它包引用这个包时，就可以通过以下引用：

```go
import myproject/mypackage
```

但你有没有想过，当别的项目需要引用你的项目中的某些包，那么就需要远程下载依赖包了，这时就需要项目的仓库地址引用，比如下面这样：

```go
import github.com/objcoding/myproject/mypackage
```

go modules 发布之后，就完全统一了包引用的地址，如上面我们说的创建 go.mod 文件后，初始化内容的第一行就是我们说的项目依赖路径，通常来说该地址就是项目的仓库地址，所有需要引用项目包的地址都填写这个地址，无论是内部之间引用还是外部引用，举个例子，goim 的内部包引用：

go.mod module：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/go9.png)

内部包引用：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/go8.png)

也即是说，在项目 启用了 go modules 之后，引用包必须跟 go mod 文件第一行包名一样，

依赖的包都会保存在 ${GOPATH}/pkg/mod 文件夹中了，我们也可以在项目底部那里查看依赖包：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/go10.png)

但也有可能会出现依赖包地址正确但会报红的情况，这时极有可能是你在 Goland 编辑器中没有将项目设置为 go modules 项目，具体设置如下：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/go11.png)

勾选了该选项之后，就会在 External Libraries 中出现 Go Modules 目录。



## 版本规则

go modules 是一个版本化依赖管理系统，版本需要遵循一些规则，比如版本号需要遵循以下格式：

```
vX.Y.Z-pre.0.yyyymmddhhmmss-abcdefabcdef
vX.0.0-yyyymmddhhmmss-abcdefabcdef
vX.Y.(Z+1)-0.yyyymmddhhmmss-abcdefabcdef
vX.Y.Z
```

vX.Y.Z 是我们仓库打的标签版本，也就是 go modules 是根据仓库标签来确定版本号的，因此我们发布版本时，需要给我们的仓库打上一个标签版本号。

也就是版本号 + 时间戳 +hash，我们自己指定版本时只需要制定版本号即可，没有版本 tag 的则需要找到对应 commit 的时间和 hash 值。

还有一个重要的规则是，版本 0 和 1，最好需要有不同的依赖路径，如：v1.0.0 和 v2.0.0 是有不同的依赖路径，下面会详细介绍一下这个版本规则。



## 发布版本

了解了 go modules 的版本规则后，现在我们发布一下该项目的版本：

```bash
$ git tag v1.0.0
$ git push --tags
```

这时我们最好还需要创建一条 v1 分支，以便我们在其它分支写代码不会影响到 v1.0.0 版本：

```bash
$ git checkout -b v1
$ git push -u origin v1
```



## 使用 module

现在我们把刚刚做好的 module，拿过来用，新建一个 gomodules 项目：

```go
package main

import (
    "fmt"
    "github.com/objcoding/testmod"
)

func main() {
    fmt.Println(testmod.SayHello("张乘辉"))
}
```

以前我们可以直接通过 go get 命令获取依赖包，但是对于一个 module 项目来说，就远远比这个有趣多了，现将项目初始化成 module 项目：

```bash
$ go mod init
```

这时 go build 等命令就会下载依赖包，并把依赖信息添加到 go.mod 文件中，同时把依赖版本哈希信息存到 go.sum 文件中：

```bash
$ go build

go: finding github.com/objcoding/testmod v1.0.0
go: downloading github.com/objcoding/testmod v1.0.0
```

这时，go.mod 文件内容如下：

```go
module gomodules
require github.com/objcoding/testmod v1.0.0
```

go.sum 文件内容如下：

```go
github.com/objcoding/testmod v1.0.0 h1:fGa15gBXoqkG0BVkQGP9i5Pg2nt8nayFpHFf+GLiX6A=
github.com/objcoding/testmod v1.0.0/go.mod h1:LGpYEmOLZhLQC3JW88STU2Ja3rsfoGZmsidsHJhDNGU=
```



这里还需要注意的是，有时候我们引用 golang.org/x/ 的一些包，但发现在伟大的天朝这个地址是被qian了，但是我们程序员也充分发挥了勤奋好学的优良传统，在 go modules 中设置了 goproxy 机制，如果 go modules 设置了代理，会优先从代理中下载依赖包，在 /etc/profile 中加入以下内容：

```bash
export GOPROXY="https://goproxy.io"
```

goproxy.io 谷歌官方的代理地址，当然还有很多国内优秀的第三方代理。

你也可以在 Goland编辑器中设置：

![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/go12.png)



## 升级版本

现在我们来升级一下 testmod 项目：

```bash
$ cd gomodules
$ echo 'package testmod
		import "fmt"
	func SayHello(name string) string {
   return fmt.Sprintf("你好, %s", name)
}' >> testmod.go
```



我把「Hello」改成「你好」，我们把这个修改在 v1 分支中进行：

```bash
$ git commit -m "update testmod" testmod.go
$ git tag v1.0.1
$ git push --tags origin v1
```

现在我们的 项目已经升级到 v1.0.1 版本了，我们可以有多种方式获取这个版本依赖，go1.11 中，go get 拥有了很多新特性，我们可以直接通过以下命令获取 v1.01 版本依赖：

```bash
$ go get github.com/objcoding/testmod@v1.0.1
```

也可以通过 go mod 命令：

```bash
$ go mod edit -require="github.com/objcoding/testmod@v1.0.1"

$ go mod tidy
```

go mod edit  -require 可以主动修改 go.md 文件中依赖的版本号，然后通过 go mod tidy 对版本进行更新，这是一条神一样的命令，它会自动清理掉不需要的依赖项，同时可以将依赖项更新到当前版本。



## 主要版本升级

上面版本规则说了，版本 0 和 1，即大版本更新，最好需要有不同的依赖路径，如：v1.0.0 和 v2.0.0 是有不同的依赖路径，那么用 go modules 怎么实现呢，我们可以通过修改 go.mod 文件第一行添加新路径：

```bash
$ cd testmod
$ echo 'module github.com/objcoding/testmod/v2' >> go.mod
```

然后我们修改 testmod 函数 Hi()：

```bash
$ cd testmod

$ echo 'package testmod
import (
	"fmt"
)
func SayHello(name, str string) string {
	return fmt.Sprintf("你好, %s, %s", name, str)
}' >> testmod.go
```

这时，SayHello() 方法将不兼容 v1 版本，我们需要新建一个 v2.0.0 版本，还是老样子，我们最好在 v2.0.0 版本新建一条 v2 分分支，将 v2.0.0 版本的代码写到这条分支中（这只是一个规范，实际上你将代码也写到任何分支中都行，Go并没有这个规范）：

```bash
$ git add *
$ git checkout -b v2
$ git commit testmod.go -m "v2.0.0"
$ git tag v2.0.0
$ git push --tags origin v2
```

这时候 testmod 的版本已经更新到 v2.0.0 版本了，该版本并不兼容以前的版本，但是目前项目依然只能获取到 v1.0.1 的依赖版本，因为我们 testmod 的 module 路径已经加上 v2 了，因此并不会出现冲突的问题，那么如果我们需要使用 v2.0.0 版本呢？我们只需要在项目中更改一下 import 路径：

```go
package main

import (
    "fmt"
  	"github.com/objcoding/testmod/v2"
)
func main() {
    fmt.Println(testmod.SayHello("张乘辉", "最近过得怎样"))
}
```

执行：

```go
go mod tidy
```

这时我们把 testmod 依赖版本号更新到了 v2.0.0 版本了，虽然是此时的 import 路径是以 “v2” 结尾，但是 Go 很人性化，我们依然可以使用 testmod 来使用。



Go 团队表示，在 Go 1.12 之前，这个特性都将会处于实验性阶段，Go 团队会努力保持兼容性。一旦模块稳定之后，对 GOPATH 的支持将会被移除掉。









