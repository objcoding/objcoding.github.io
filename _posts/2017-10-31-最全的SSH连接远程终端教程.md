---
layout: post
title: "最全的SSH连接远程终端教程"
categories: Linux
tags: ssh
author: zch
---

* content
{:toc}
现在工作上有大部分时间都在终端上，需要经常在终端部署项目，查看日志，找bug，所以写一篇ssh连接远程终端的文章，以此记录一下整个配置过程，因为自己也是一个健忘的人，在此过程中也涉及了一些linux权限的知识点。









> SSH 分为客户端和服务端。 
> 服务端是一个守护进程，一般是 sshd 进程，在后台运行并响应来自客户端的请求。提供了对远程请求的处理，一般包括公共密钥认证、密钥交换、对称密钥加密和非安全连接。 
> 客户端一般是 ssh 进程，另外还包含 scp、slogin、sftp 等其他进程。



## 客户端生成公钥密钥

用git命令ssh-keygen -t rsa ，会在~/下生成一个.ssh的隐藏文件夹，里面包含了id_rsa密钥和id_rsa.pub公钥，等下把公钥添加到服务器。

## 下载ssh，配置ssh，启动sshd

```bash
yum install openssh-server -y
```

OpenSSH的主配置文件：/etc/ssh/sshd_config

以下是一些常用设置：

```bash
# 设置SSH的端口号是22(默认端口号为22)
Port 22

# 使用ssh验证登陆
RSAAuthentication yes 
pubkeyAuthentication yes

# 公钥文件路径
AuthorizedKeysFile	.ssh/authorized_keys

# 禁止密码登陆
PasswordAuthentication no


```

开启sshd：

```bash
service ssh start #启动
service ssh stop #停止
service ssh restart #重启 
```

查看进程：

```bash
ps -ef|grep ssh
```



## 创建用户目录，添加公钥，配置权限

在ssh启动后，会在～/下创建一个.ssh隐藏文件夹，里面有一个authorized_keys文件，**可以在这个文件添加需要连接的服务器的客户端的公钥，但是一般不会这么做，这会有安全隐患，因为在root目录下的公钥的客户端登陆到服务器后会直接取得root权限，所以我会在/home目录下创建一个以客户名称的用户目录，再把～/.ssh文件夹复制到该用户目录下**，这里需要一些权限操作：

将.ssh文件夹设置成只有属主有读、写、执行权限：

```bash
chmod 700 /home/zhangch/.ssh
```

再将authorized_keys文件设置成只有属主有读写权限：

```bash
chmod 600 /home/zhangch/.ssh/authorized_keys
```

最后还需要将.ssh文件夹里的文件设置成该用户的所有者和所属的组：

```bash
chown zhangch|zhangch*
```

如下：

![ssh](https://raw.githubusercontent.com/zchdjb/zchdjb.github.io/master/images/ssh.jpg)





## 连接终端，获取root权限密码设定

在客户端～/.ssh里面创建一个config文件：

```bash
mkdir config
```

编辑：

```bash
sudo vim config
```

添加内容：

```bash
# 测试服务器
Host test
    HostName xxx.xx.xxx.xxx #服务器ip地址
    Port 22 #服务器配置的ssh端口号
    User zhangch #在服务器的用户名（对应用户文件夹名字）
```

然后在终端(macOS推荐使用iTerm2)输入：

```bash
ssh test
```

到这里，就可以登上服务器了，但现在你还没获得root权限。

接下来就是给用户配置需要输入密码获取root权限的操作：

在服务器root权限下给zhangch用户添加密码：

```bash
passwd zhangch
```

然后就是输入密码

这时还需要在/etc/sudoers给该用户临时提升权限（sudo就是我们常用的命令，仅需要输入当前用户密码，便可以完成权限的临时提升）

```bash
sudo vim /etc/sudoers
```

添加下面内容：

```bash
# 格式为（用户名    网络中的主机=（执行命令的目标用户）    执行的命令范围）
user    ALL=(ALL)       ALL
```

这时候退出保存可能会遇到文件只读状态，我们还需要给该文件更改权限：

```bash
chmod 700 /etc/sudoers
```

在登陆服务器之后，需要取得临时root权限：

```bash
sudo su -
```

提示你输入密码，输入刚刚的密码，这时候你就拥有了root权限了。为安全起见，记得操作完后切换回用户目录：

```bash
su - zhangch
```

完。





附：

```bash
-rw------- (600) -- 只有属主有读写权限。

-rw-r--r-- (644) -- 只有属主有读写权限；而属组用户和其他用户只有读权限。

-rwx------ (700) -- 只有属主有读、写、执行权限。

-rwxr-xr-x (755) -- 属主有读、写、执行权限；而属组用户和其他用户只有读、执行权限。

-rwx--x--x (711) -- 属主有读、写、执行权限；而属组用户和其他用户只有执行权限。

-rw-rw-rw- (666) -- 所有用户都有文件读、写权限。这种做法不可取。

-rwxrwxrwx (777) -- 所有用户都有读、写、执行权限。更不可取的做法。

```

