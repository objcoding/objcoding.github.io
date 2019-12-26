---
layout: post
title: "Docker网络模型"
categories: Docker
tags: swarm network overlay
author: zch
---

* content
{:toc}
单机容器内的通信是通过 docker 自带的网桥连接互通的，如果是集群，那么做这些单机网络模型就行不通了，因为集群必然会将一个服务的多个任务需要分布到不同的机器上进行部署，因为如果把所有的任务都分配到一台机器部署了，这台服务器有故障，就会导致这个服务不能正常运行了，而且一个集群内，不同主机之间的容器是怎么进行通信的呢，这里我们就涉及到 docker 网络模型。













## 容器的端口映射

还记得 docker 单机部署时的 run -p port:port 吗？这样做的目的是将 docker 容器内的端口映射到宿主机的端口上，以便能够通过外网 ip 访问到 docker 容器，这时我们就想，如果我们把所有容器的接口都暴露在宿主机中，通过访问外网 ip 来达到容器间通信，这不是万事大吉了吗？

但是在集群中如果这样暴露容器的端口，是有问题的，如果其中一个容器监听了宿主机 8080 端口，那么其他容器只能映射到其它端口了，因为端口并不能被共享，而且映射到宿主机的端口上，意味着容器也就暴露到外网了，如果要限制访问，那么就需要做一些安全配置。



## 单机网络模型

在介绍跨主机网络模型前，先来看看单机网络模型，在安装 docker 之后，docker 就会有 4 种网络模型，分别是：

- host 模式，使用 --net=host 指定。
- container 模式，使用 --net=container:NAME_or_ID 指定。
- none 模式，使用 --net=none 指定。
- bridge 模式，使用 --net=bridge 指定，默认设置。

但这四种网络模式都仅限于单机，其中 bridge 网络模型是 docker 的默认单机网络模型，它会将一个主机上的 docker 容器连接到一个虚拟网桥上，这个虚拟桥名称为 docker0，如下图：

![brige](https://gitee.com/objcoding/md-picture/raw/master/img/d_network1.png)

单机中的容器之间就可以通过 docker0 互相通信了，但是如果容器被分布在不同主机上，在没有跨主机网络模型前，只能通过映射端口的形式来通信了。

![brige](https://gitee.com/objcoding/md-picture/raw/master/img/d_network2.png)

如上图，net1 和 net2 都代表一台主机中的 docker0 网络，在同主机下的容器通过 docker0 网络互相通信，但是在不同主机中却又是隔离的。



## 跨主机网络模型

docker 1.9 版本之后，加入了一个默认的 overlay 的网络模型，它是 docker swarm 内置的跨主机通信方案，这是一个基于 vxlan 协议的网络实现，其作用是虚拟出一个子网，让处于不同主机的容器能透明地使用这个子网。所以跨主机的容器通信就变成了在同一个子网下的容器通信，看上去就像是同一主机下的 bridge 网络通信。

关于详细的 vxlan 协议原理，请移步：[vxlan 协议原理简介](http://cizixs.com/2017/09/25/vxlan-protocol-introduction)



在 swarm 管理节点发布的服务想要监听端口，只需要在 像 docker run 一样在后缀加 -p 8080:8080 就可以了，如下：

```bash
$ docker service create --replicas 2 -p 8080:8080 --name hello \
chenghuizhang/helloword:0.0.2
```

但是跟 docker run 的 -p 又有本质的区别，实际上面那条命令并没有将 8080 端口直接暴露出去，而是将 8080 端口托付给 docker 的 overlay 网络模型中了。

![docker ps](https://gitee.com/objcoding/md-picture/raw/master/img/d_network3.png)

如上图可知，hello 服务的两个实例都在同一台服务器，都是 8080 端口，且没有映射到宿主机的端口上。

查看 docker 默认网络：

```bash
$ docker network ls
```

![docker network](https://gitee.com/objcoding/md-picture/raw/master/img/d_network5.png)

其中 ingress 为 docker 默认的 overlay 网络。

查看 ingress 网络信息：

```bash
$ docker network inspect ingress
```

```
[
    {
        "Name": "ingress",
        "Id": "x58u3pdo4z4qooriloi82l58k",
        "Created": "2018-08-15T12:54:22.080771222+08:00",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "10.255.0.0/16",
                    "Gateway": "10.255.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "aef3e18d73b1db723aab0f162600a805aa7c50b5f51ec31d82d80b7eb3438c08": {
                "Name": "hello.1.tzk61cn6d0jjat2w4mu5x8bbf",
                "EndpointID": "40e9352419b4eccea739be3a6ab7e6327ee28a2220ce185750f028d74e2e05c8",
                "MacAddress": "02:42:0a:ff:00:05",
                "IPv4Address": "10.255.0.5/16",
                "IPv6Address": ""
            },
            "ef4a7f16567d3b51d0dff629b3b6252f50d06ec77a24b36dcd8d40b6ab9afc2d": {
                "Name": "hello.2.lcvumjyd19tymzankbjwita1w",
                "EndpointID": "d5b9a2e41654097337d6f22b139e1501b253e260d2ac3ac52e8eaf491a70340d",
                "MacAddress": "02:42:0a:ff:00:06",
                "IPv4Address": "10.255.0.6/16",
                "IPv6Address": ""
            },
            "ingress-sbox": {
                "Name": "ingress-endpoint",
                "EndpointID": "b610f32940e6066751a6c7b8d87f11874077f779a00d28855d8d3329d15783e9",
                "MacAddress": "02:42:0a:ff:00:03",
                "IPv4Address": "10.255.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4096"
        },
        "Labels": {},
        "Peers": [
            {
                "Name": "VM_0_10_centos-8ef37c047944",
                "IP": "172.16.0.10"
            }
        ]
    }
]
```

由于 orverlay 网络模型是基于 vxlan 协议的网络实现，所以根据上面的网络信息可知，它是要在三层网络中虚拟出二层网络，即跨网段建立虚拟子网，也就是把 docker 要发送的信息先发送到虚拟子网地址 10.255.0.1，再由虚拟子网包装为宿主机的真实网网址  172.16.0.10，这样做的好处就是不会公开暴露容器的端口，让这些事情交给  overlay 网络驱动去做就行了，而且在同一台服务器，不会引起端口冲突，最重要的一点是可以实现集群容器间的负载均衡。

![overlay network](https://gitee.com/objcoding/md-picture/raw/master/img/d_network6.png)

**正如它的名字一样，在所有容器的上面一层，覆盖了一层网络，该网络可以使在集群中的容器像本地通信一样，所以 orverlay 网络模型也称之为覆盖网络。**



## 构建自定义 overlay 网络集群

- 新建网络驱动：

```bash
$ docker network create -d overlay mynet
```

-d 指定 mynet 网络驱动为 overlay 类型。

- 创建 etcd 服务：

```bash
$ docker service create \
--name etcd \
--replicas 1 \
--network mynet \
-p 2379:2379
```

- 创建 mysql 服务：

```bash
$ docker service create \
--name mysql-galera \
--replicas 3 \
--network mynet \
-p 3306:3306
```

- 创建 hello 应用服务：

```bash
$ docker service create \
--name hello chenghuizhang/helloword:0.0.2 \
--replicas 2 \
--network mynet \
-p 8080:8080
```

到这里，我们已经构建了一个名为 mynet 的网络集群了，集群网络模型如下：

![swarm network model](https://gitee.com/objcoding/md-picture/raw/master/img/d_network4.png)

swarm 集群的内部会为容器的各个节点之间负责负载均衡的管理，无需我们去操心了，如上如图三台服务器，无论我们访问的哪台服务器，都可以访问到 docker 各个可用节点中，比如访问 172.16.1.11:8080，也可以通过 swarm 集群的负载均衡转发到 172.16.1.12:8080。
