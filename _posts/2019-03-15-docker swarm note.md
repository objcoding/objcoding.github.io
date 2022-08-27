---
layout: post
title: "Docker Swarm 集群部署笔记"
categories: Docker
tags: docker
author: 张乘辉
---

* content
{:toc}






## Docker Swarm 集群的一些概念

### 节点

swarm集群分为管理节点和工作节点，管理节点可以操作swarm命令控制swarm集群，工作节点是用于运行服务的节点，理论上管理节点也可以是工作节点，一样可以用于运行服务。

一般来说一个swarm集群需要两个以上的管理节点。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/docker16.png)



### 服务

在分布式集群应用中，应用的不同部分拆分成“服务”，服务在swarm集群中可部署在多个节点上，形成集群，可使用swarm命令动态扩展服务在swarm集群中运行的实例数量，以满足需求。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/docker17.png)



### 技术栈

技术栈是一组相关的服务，它们共享依赖项并且可以一起进行编排和扩展，比如我们的vipay和cash项目的各个服务，可使用compose.yml文件编排成vipay技术栈以及cash技术栈，并使用 `docker stack deploy`分别进行部署。技术栈也是swarm集群中层次结构的最高级别。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/docker18.png)





## Docker Swarm 集群的命令栈

docker swarm: 集群管理，子命令有 init, join, leave, update

docker service: 服务管理，子命令有 create, inspect, update, remove, tasks

docker node：节点管理，子命令有accept, promote, demote, inspect, update, tasks, ls, rm

docker network: 网络管理,子命令有connect,create,disconnect,inspect,ls,prune,rm

docker stack: 服务上层管理，子命令有deploy,ls,ps,rm,services

docker volume：数据卷管理，子命令有ls,inpsect,rm,create



以下是一些常用的命令操作：

```bash
# 创建服务
docker service create \  
  --image nginx \
  --replicas 2 \
  nginx 

# 更新服务
docker service update \  
  --image nginx:alpine \
  nginx 

# 删除服务
docker service rm nginx

# 减少服务实例(这比直接删除服务要好)
docker service scale nginx=0

# 增加服务实例
docker service scale nginx=5

# 查看所有服务
docker service ls

# 查看服务的容器状态
docker service ps nginx

# 查看服务的详细信息。
docker service inspect nginx
```





## 创建 Docker Swarm 集群步骤

### 安装docker

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

### 配置daemo.json

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

### 启动docker

```bash
$ sudo systemctl start docker
```

### 初始化一个swarm集群（后续添加节点该步骤省略）

```bash
$ sudo docker swarm init
```

### 节点加入集群

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





## 创建 Docker 原生私有库

使用docker官方registry仓库镜像，直接在仓库服务器pull下镜像，docker run就可以创建一个私有仓库了

docker默认推送到私有库只能是用https协议，在原有的基础上需要配置一个拥有权限认证的私有仓库：

```bash
# 创建仓库数据卷
$ sudo docker volume create registry

$ sudo docker run -d -p 5000:5000 \
-v registry:/var/lib/registry \
--restart=always \
--name registry registry
```

在`/etc/docker/daemon.json`中加入以下内容：

```json
{
  "registry-mirror": [
    "https://registry.docker-cn.com"
  ],
  "insecure-registries": [
    "仓库内网ip:端口"
  ]
}
```

在每个节点添加兼容私有仓库非 https 协议配置：

```bash
$ vim /etc/sysconfig/docker

OPTIONS='--insecure-registry 192.168.1.111:5000'
```





## Docker Swarm 集群的可视化管理

在swarm集群中添加portainer可视化管理工具，先下载compose编排文件：

```bash
$ curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
```

稍微修改一下文件，以适应我们的swarm集群，比如网络驱动，容器名称，端口等等。

deploy运行：

```bash
$ sudo docker stack deploy --compose-file=portainer-agent-stack.yml portainer
```





## Docker Swarm 集群的负载均衡

单机模型：

同一主机docker容器间通过docker内置的虚拟网桥docker0通信, 如果需要跨主机通信, 那么就通过端口映射的方式.

跨主机模型：

通过vxlan网络协议实现, 简单来说就是在所有容器的上面一层，覆盖了一层网络，该网络可以使在集群中的容器像本地通信一样，所以 orverlay 网络模型也称之为覆盖网络, 容器本身并没有把端口映射到主机, 而是将端口暴露的事情交给覆盖网络去处理了.

docker的覆盖网络有个好处就是在集群下, 通过任意一个节点可以访问到对应的服务, 即使当前节点没有该服务实例, 这样也间接性地实现了节点间的负载均衡.

创建跨主机网络驱动：

```bash
$ sudo docker network create -d overlay mynet
```



## Swarm集群服务的更新与版本回滚

更新执行命令: docker service update --images xxx:latest my_project

回滚执行命令: docker service update --rollback my_project

指定回滚版本号：docker service update --images xxx:latest my_project:<上一个版本>



## 镜像版本的一些规范

每次需要打包构建镜像名称：<仓库地址>/<服务名称>:<分支>-<时间戳>

再打包一个镜像名称：<仓库地址>/<服务名称>:<分支>-latest

这么做的好处是：

有时间戳的镜像版本作为回滚的作用，可通过命令 docker service update images 命令回滚到任意一个版本

无时间戳的镜像版本为当前运行的最新版本。每次镜像更行构建完后，默认运行该镜像。



## 使用 docker swarm 集群的好处

### 1.可动态调整服务的实例个数

当我们需要增加一个服务部署的实例个数时，我们不需要重新在一台机器里面做一些重复劳动性的工作了，我们只需动动手指头，就可以动态扩。

我直接可通过 docker swarm集群的管理界面工具上，找到相关服务，手动调整实例个数就ojbk了，当然你想逼格更高点，你直接去管理节点敲命令行也是ojbk的：

```
$ sudo docker service scale myService = 数量
```

我们以后就再也不用关心项目部署在哪台机了，它会自动随机分配部署到集群的任意一个节点，我们只需通过swarm集群，就可负载均衡地随机访问到任意一个实例。

### 2.可动态扩容

当我们集群内集群负载过高时，可以增加若干台机器，在每台加入机器装上docker，执行以下加入集群的命令，就可以加入集群，听从管理节点分配的工作。完全不需要在新增的机器上面做一些重复性劳动，你只需要安装docker，就这么任性。

```
docker swarm join --token SWMTKN-1-69luztakii9ix7f5osezl0v6l2ibfzp1vqc0gbhcous63hm1fx-8p3vxanj97f2e0jflznihvl8f 172.17.10.127:2377
```

### 3.一次打包，到处运行

这个也是docker官方的宣传口语，我们只需将所有的运行时依赖打包成一个镜像，就可以任性地到处运行了。测试运维小伙伴再也不需要重新将环境搭建一次了，人都会犯错的，你不能保证你搭建的环境跟我开发的环境是一致的，有时候就会出现我在sit环境部署的很好，一上uat就变火葬场的情况。用上docker，将会大大杜绝这种事情的发生。

### 4.自动化发版

jenkins监听到仓库对应分支的代码有改动，自动拉取代码，制作新镜像，执行远程命令：

```
$ sudo docker update --images <imagesname> <servicename>
```

将会自动更新该服务所有的实例，且不需要停机停服更新，完全可实现平滑升级，比如该服务有3个实例，那么可设置依次更新。



