---
layout: post
title: "Docker Swarm集群的负载均衡"
categories: Docker
tags: swarm loadbalance overlay
author: zch
---

* content
{:toc}
swarm 集群的内部会为容器的各个节点之间负责负载均衡的管理，现在我们来验证一下 swarm 的负载均衡特性。

















1. 任意一台服务器访问服务
2. Consul 服务发现
3. 一台没有改服务实例的服务器访问该服务端口
4. 查看工作节点的network列表是否与管理节点同步（只需在manager节点创建，当有Service连接该overlay网络时，将会自动在所分配的worker节点上自动创建该overlay网络。）
5. **原来不同内网环境需要指定外网ip才能负载均衡 docker swarm join —token —advertise-addr 外网ip**
6. lsof -i:8080 查看没有服务的服务器是否有监听该服务的端口
7. 需要防火墙开放特定端口