---
layout: post
title: "Docker Stack多服务编排"
categories: Docker
tags: compose swarm stack
author: zch
---

* content
{:toc}
之前 swarm 集群中`docker service create`一次只能部署一个微服务，我们可以使用 docker compose 一次启动多个服务。











- 编写 docker-compose.yml 文件：

```bash
version: "3"

services:
  go-gin-demo:
    image: chenghuizhang/go-gin-demo:v3
    ports:
      - 8081:8081
    networks:
      - overlay
    deploy:
      mode: replicated
      replicas: 2
  hello:
    image: chenghuizhang/helloword:0.0.2
    ports:
      - 8080:8080
    networks:
      - overlay
    deploy:
      mode: replicated
      replicas: 2

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - 8090:8080
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
             
networks:
  overlay:
```

该 compose 文件制定部署 3 个服务，分别指定了服务的端口、服务实例个数、网络、镜像名称等等， 其中的  visualizer 服务提供一个可视化页面，我们可以从浏览器中很直观的查看集群中各个服务的运行节点。



- 部署

```bash
$ docker stack deploy -c docker-compose.yml mynet
```

现在我们打开浏览器输入 任一节点 IP:8090 即可看到各节点运行状态。如下图所示：

![visualizer](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/docker_compose.png)

也可以在服务器里面查看服务运行情况：

```bash
$ docker stack ps mynet
```

![docker compose](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/docker_compose.png)

**stack 是一组相互关联的服务，也就是它是服务的上一层，这些服务共享依赖关系，并且可以一起编排和缩放。单个 stack 能够定义和协调整个应用程序的功能（尽管非常复杂的应用程序可能希望使用多个堆栈），简单来说 stack就是一组服务的集合。**

stack 相关命令：

```
deploy      Deploy a new stack or update an existing stack
ls          List stacks
ps          List the tasks in the stack
rm          Remove one or more stacks
services    List the services in the stack
```

现在我们更新一下 docker-compose.yml 文件，增加 portainer 服务：

```
portainer:
    image: portainer/portainer
    ports:
      - "9000:9000"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
```

```bash
$ docker stack deploy -c docker-compose.yml mynet
```

打开页面：

![docker stack](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/docker_stack.png)

portainer 是 docker swarm 集群容器管理页面，可管理 Docker 容器、image、volume、network 等，当让我们还可以在其页面上添加多个stack：

![docker stack](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/docker_stack2.png)