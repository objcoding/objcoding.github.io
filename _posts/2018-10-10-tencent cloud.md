---
layout: post
title: "腾讯云重启后登录不了解决办法"
categories: Linux
tags: 腾讯云
author: 张乘辉
---

* content
{:toc}
今天闲的蛋疼把腾讯云服务器重启了，发现登陆不了，由于没有禁止防火墙开机启动，重启后导致防火墙重启启动，导致 SSH 端口变成了过滤状态。











首先进入腾讯云控制台->云服务器->安全组，新建安全组，这里为了方便，生成了一个开发全部端口的安全组，如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/tencent_cloud.png)





切换到云主机tab，找到对应服务器，点击右边「更多」按钮，然后配置安全组，选择刚刚新建好的安全组。

![](https://gitee.com/objcoding/md-picture/raw/master/img/tencent_cloud_3.png)



腾讯云工程师告诉我，只需要等待10来分钟就生效了，但是等待了半个小时后，发现还是登陆不了，于是我告知腾讯云工程师，答复如下：

![](https://gitee.com/objcoding/md-picture/raw/master/img/tencent_cloud_4.png)



于是按照上述方法，找到登陆按钮，点击弹出登陆框：

![](https://gitee.com/objcoding/md-picture/raw/master/img/tencent_cloud_5.png)

这里我们需要把登陆框关闭，选择其它登陆方式：

![](https://gitee.com/objcoding/md-picture/raw/master/img/tencent_cloud_6.png)

把弹出框拉到最下，选择 VNC 方式登录，这时就会出现登陆 Linux 界面了，输入账号密码，然后执行如下操作：

```bash
$ systemctl stop firewalld.service // 停止 firewalld
$ systemctl disable firewalld.service // 禁止firewall开机启动
```

