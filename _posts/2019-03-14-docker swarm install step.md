---
layout: post
title: "创建Docker Swarm集群步骤"
categories: Docker
tags: swarm
author: zch
---

* content
{:toc}
创建一个Docker Swarm集群很简单，如果在大规模集群的项目，可以使用Docker Machine批量创建与管理节点，我这里就简单记录一下搭建几个节点的swarm集群的步骤。







## 安装docker

设置docker版本镜像仓库，从而可以轻松完成安装和升级任务：

```bash
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

添加Docker源，始终需要使用stable镜像仓库进行更新docker版本：

```bash
$ sudo yum-config-manager \
--add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

执行安装：

```bash
sudo yum makecache
```

安装Docker：

```bash
$ sudo yum install docker-ce
```

如果出现以下报错：

```
Downloading packages:
warning: /var/cache/yum/x86_64/7/base/packages/wget-1.14-13.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Retrieving key from http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
 
The GPG keys listed for the "CentOS-7 - Base - 163.com" repository are already installed but they are not correct for this package.
Check that the correct key URLs are configured for this repository.
 
 Failing package is: wget-1.14-13.el7.x86_64
 GPG Keys are configured as: http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
```

解决方法：

```bash
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```





## 配置daemo.json

```bash
$ vim /etc/docker/daemon.json
```

```json
{
  "registry-mirror": [
    "https://registry.docker-cn.com"
  ],
  "insecure-registries": [
    "172.17.10.127:5000"
  ]
}
```

以上配置目的添加一个私有库以及镜像加速器。



## 启动docker

```bash
$ sudo systemctl start docker
```



## 初始化一个swarm集群（后续添加节点该步骤省略）

```bash
$ sudo docker swarm init
```



## 节点加入集群

查看使用主节点的token添加工作节点到集群的命令：

```bash
$ sudo docker swarm join-token worker
```

查看使用主节点的token添加管理节点到集群的命令：

```bash
$ sudo docker swarm join-token manage
```

集群中加入一个节点：

```bash
$ sudo docker swarm join \
--token SWMTKN-1-69luztakii9ix7f5osezl0v6l2ibfzp1vqc0gbhcous63hm1fx-8p3vxanj97f2e0jflznihvl8f \
<HOSST>:<NAME>
```