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









## 创建测试项目

- 编写测试程序：

```go
func main() {
  addrs, err := net.InterfaceAddrs()
  if err != nil {
    panic(err)
  }
  r := gin.Default()
  gin.Logger()
  r.GET("/addrs", func(c *gin.Context) {
    c.JSON(200, gin.H{
      "addrs": addrs,
    })
  })
  r.Run(":8081")
}
```

- 编写Dockerfile：

```dockerfile
FROM golang:latest

WORKDIR $GOPATH/src/go-gin-demo

COPY . $GOPATH/src/go-gin-demo

RUN go get github.com/gin-gonic/gin && go build .

EXPOSE 8081

ENTRYPOINT ["./go-gin-demo"]

```

- 打包镜像并上传到 docker hub：

```bash
$ docker build -t chenghuizhang/go-gin-demo:v2 .
$ docker push chenghuizhang/go-gin-demo:v2
```





## 创建集群

首先初始化一个管理节点：

```bash
$ docker swarm init --advertise-addr 193.xxx.61.178
```

这里需要说明一下，由于我的两台服务器都同于一个内网环境，所以这里需要指定外网 ip，得到以下命令：

```bash
$ docker swarm join --token xxxxxxxxxxxxxxxx 193.xxx.61.178:2377
```

另一台服务器加入，现在得到了拥有两个节点的 swarm集群：

![docker swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm9.png)

这里特别注意一下，由于是



## 部署测试

- 创建集群网络驱动：

```bash
$ docker network create -d overlay mynet
```

- 部署 go-gin-demo 到其中一个节点，另外一个节点是否可通过 swarm 负载均衡访问：

```bash
$ docker service create -p 8081:8081 --network mynet --replicas 1 --name go-gin-demo chenghuizhang/go-gin-demo:v2
```











1. 创建测试项目（打包部署）
2. 创建集群
3. 访问测试(部署一个实例时，同时部署两个实例时)







1. 任意一台服务器访问服务
2. 一台没有改服务实例的服务器访问该服务端口
3. 查看工作节点的network列表是否与管理节点同步（只需在manager节点创建，当有Service连接该overlay网络时，将会自动在所分配的worker节点上自动创建该overlay网络。）
4. **原来不同内网环境需要指定外网ip才能负载均衡 docker swarm join —token —advertise-addr 外网ip**
5. lsof -i:8080 查看没有服务的服务器是否有监听该服务的端口
6. 需要防火墙开放特定端口