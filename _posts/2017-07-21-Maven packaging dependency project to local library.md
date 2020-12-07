---
layout: post
title: "Maven打包依赖项目到本地库"
categories: Tools
tags: Maven
author: 张乘辉
---

* content
{:toc}
在分布式系统下，我们会把工具库单独写成一个独立的项目，此时其它系统需要依赖它，那么我们就需要把它打包成可执行的jar包，并安装到本地仓库中。









现在有个叫server-common-toolbox的工具项目，maven依赖信息如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/maven4.png)

首先进入项目所在目录，然后执行mvn install命令：

![](https://gitee.com/objcoding/md-picture/raw/master/img/maven1.png)

如果出现一下信息，就说明打包安装成功了：

![](https://gitee.com/objcoding/md-picture/raw/master/img/maven2.png)

打包成功后，可在本地仓库查看：

![](https://gitee.com/objcoding/md-picture/raw/master/img/maven3.png)

接下来就可以在各个子系统中添加依赖了：

![](https://gitee.com/objcoding/md-picture/raw/master/img/maven5.png)