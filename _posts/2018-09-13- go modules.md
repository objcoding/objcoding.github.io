---
layout: post
title: "go modules详解"
categories: Go
tags: module
author: zch
---

* content
{:toc}
我们以前用 go get 获取依赖其实是有潜在危险的，因为我们不确定最新版依赖是否会破坏掉我们项目对依赖包的使用方式，即当前项目可能会出现不兼容最新依赖包的问题。随着 go1.11 的发布，go 给我们带来了 module 新特性，这是 Go 语言新的一套依赖管理系统。











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
	func Hi(name string) string {
   return fmt.Sprintf("Hi, %s", name)
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

测试项目地址：[testmod](https://github.com/objcoding/testmod)

## go mudules 版本规则

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

这时我们最好还需要创建一条 v1 分支，szzsz以便我们在其它分支写代码不会影响到 v1.0.0 版本：

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
    fmt.Println(testmod.Hi("zch"))
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

这时，go.md 文件内容如下：

```
module gomodules
require github.com/objcoding/testmod v1.0.0
```

go.sum 文件内容如下：

```
github.com/objcoding/testmod v1.0.0 h1:fGa15gBXoqkG0BVkQGP9i5Pg2nt8nayFpHFf+GLiX6A=
github.com/objcoding/testmod v1.0.0/go.mod h1:LGpYEmOLZhLQC3JW88STU2Ja3rsfoGZmsidsHJhDNGU=
```



## 升级版本

现在我们来升级一下 testmod 项目：

```
$ cd gomodules
$ echo 'package testmod
		import "fmt"
	func Hi(name string) string {
   return fmt.Sprintf("Hi, %s!", name)
}' >> testmod.go
```

我们姑且就在打印语句上面加个感叹号，我们把这个修改在 v1 分支中进行：

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
	"errors"
)
func Hi(name, lang string) (string, error) {
	switch lang {
	case "en":
		return fmt.Sprintf("Hi, %s!", name), nil
	case "pt":
		return fmt.Sprintf("Oi, %s!", name), nil
	case "es":
		return fmt.Sprintf("¡Hola, %s!", name), nil
	case "fr":
		return fmt.Sprintf("Bonjour, %s!", name), nil
	case "cn":
		return fmt.Sprintf("你好，%s！", name), nil
	default:
		return "", errors.New("unknown language")
	}
}' >> testmod.go
```

这时，Hi() 方法将不兼容 v1 版本，我们需要新建一个 v2.0.0 版本，还是老样子，我们最好在 v2.0.0 版本新建一条 v2 分分支，将 v2.0.0 版本的代码写到这条分支中（这只是一个规范，实际上你将代码也写到任何分支中都行，go并没有这个规范）：

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
    fmt.Println(testmod.Hi("zch", "en"))
}
```

执行：

```
go mod tidy
```

这时我们把 testmod 依赖版本号更新到了 v2.0.0 版本了，虽然是此时的 import 路径是以 “v2” 结尾，但是 go 很人性化，我们依然可以使用 testmod 来使用。



## go modules 命令大全

go modules 目前暂时有如下命令：

```
Usage:

	go mod <command> [arguments]

The commands are:

	download    download modules to local cache (下载依赖的module到本地cache))
	edit        edit go.mod from tools or scripts (编辑go.mod文件)
	graph       print module requirement graph (打印模块依赖图))
	init        initialize new module in current directory (再当前文件夹下初始化一个新的module, 创建go.mod文件))
	tidy        add missing and remove unused modules (增加丢失的module，去掉未用的module)
	vendor      make vendored copy of dependencies (将依赖复制到vendor下)
	verify      verify dependencies have expected content (校验依赖)
	why         explain why packages or modules are needed (解释为什么需要依赖)
```



## 总结

go 团队表示，在 go 1.12 之前，这个特性都将会处于实验性阶段，go 团队会努力保持兼容性。一旦模块稳定之后，对 GOPATH 的支持将会被移除掉。

如果 go 之后还有什么更新，我也会在这里对 go modules 的最新玩法进行同步跟新目前暂时发现 goland  IDE 对 go modules 的支持还不是很给力啊，不知道是 go modules 的原因还是 IDE 的原因，为此我在「go 语言中文网」发了一个问答主题：

[关于go modules的问题](https://studygolang.com/topics/6496#reply0)









