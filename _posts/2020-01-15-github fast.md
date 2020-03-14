---
layout: post
title: "彻底解决 GitHub 拉取代码网速慢的问题"
categories: GitHub
tags: 
author: zch
---

* content
{:toc}
本人重度依赖 GitHub，面向 GitHub 编程，GitHub 可以让我每天早上打开电脑，假装了解最新开源项目。

最近你们有没有发现，GitHub 明显变慢了，如果没有 fanqiang，拉取代码的速度简直惨不忍睹，如果拉取的量少还可以勉强拉下来，但是遇到数据量大的时候，2 KiB/s 的速度你能忍？拉到中途超时就让你痛不欲生。

最近我就遇到这个问题，seata 社区的 seata.github.io 仓库有阵子突然增加了好多数据，我发现我已经拉不下来了，这时可以利用 Gitee 作为中间代理，下面详细说说具体操作过程。









在 GitHub 中，一共有两个仓库：

1. **seata**：Github 的 Seata 主仓库为：https://github.com/seata/seata.github.io.git
2. **objcoding**：我从 Seata 主仓库中 fork 过来一个仓库，地址为：https://github.com/objcoding/seata.github.io.git

以下内容将用 seat、objcoding 表示这两个仓库。



Gitee 创建仓库时，可以导入已有仓库时选择从 GitHub 仓库中导入，这时我们填写 Seata 主仓库地址，意味着 Gitee 仓库将可以从 Seata 主仓库中同步代码 ：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200115202001.png)

将 Gitee 仓库 clone 到本地（此时仓库名称默认 origin）：

```bash
git clone https://gitee.com/objcoding/seata.github.io.git
```

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200115204049.png)

这个速度快到我想哭，你能想象GitHub 2 KiB/s 的悲惨人生么。

添加 objcoding 远程仓库：

```bash
git remote add objcoding https://github.com/objcoding/seata.github.io.git
```

fetch objcoding 远程仓库内容到本地：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200115204257.png)

**速度很快，因为远程仓库中的绝大部份代码，已经从 gitee 拉取下来了。**

添加 seata 远程仓库：

```bash
git remote add seata https://github.com/seata/seata.github.io.git
```

同理，fetch seata 远程仓库内容到本地。

这时候，我本地仓库就拥有了三个远程仓库了，分别是：

1. origin：码云仓库，该仓库可以从 seata 仓库中同步代码；
2. objcoding：从 seata 仓库中 fork 的仓库；
3. seata：seata 主仓库。

为什么这里还需要添加 seata 仓库呢？这是因为一般来说，seata 主仓库增加的代码数据量都很少，即使是 2Kib/s 的速度，也是可以拉取下来的，所以平时可以直接从 seata 主仓库中拉取最新代码就可以了，但是像 seata.github.io 仓库，突然某个大佬上传了几十兆数据，那么此时我就可以利用 Gitee 仓库去同步这些代码，具体操作如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200115202542.png)

接下来 fetch gitee 对应的分支，就可以将这些数据拉取下来了。

![](https://gitee.com/objcoding/md-picture/raw/master/img/20200115214037.png)

以上是整个同步过程分析。

