---
layout: post
title: "验证Docker Swarm集群的负载均衡"
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
  resp, _ := http.Get("http://myexternalip.com/raw")
  defer resp.Body.Close()
  content, _ := ioutil.ReadAll(resp.Body)
  r := gin.Default()
  r.GET("/addr", func(c *gin.Context) {
    c.JSON(200, gin.H{
      "addr": string(content),
    })
  })
  r.Run(":8081")
}
```

- 编写 Dockerfile：

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
$ docker build -t chenghuizhang/go-gin-demo:v3 .
$ docker push chenghuizhang/go-gin-demo:v3
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

另一台服务器加入，现在得到了拥有两个节点的 swarm 集群：

![docker swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm9.png)

**这里特别注意一下，由于是加入管理节点需要通过外网，所以`docker swarm join`加个地址参数：**

```bash
$ docker swarm join --token xxxxxxxxxxxxxxxx 193.xxx.61.178:2377 --advertise-addr 111.xxx.254.127
```





## 部署测试

- 创建集群网络驱动：

```bash
$ docker network create -d overlay mynet
```

- 部署 go-gin-demo 到其中一个节点，另外一个节点是否可通过 docker 的 overlay 跨主机网路驱动访问：

```bash
$ docker service create -p 8081:8081 --network mynet --replicas 1 --name go-gin-demo chenghuizhang/go-gin-demo:v3
```

查看服务：

```
$ docker service ps go-gin-demo 
```

发现 go-gin-demo 部署到工作节点了，这时我们通过管理节点 ip 访问，结果如下：

![docker swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/swarm10.png)

说明即使管理节点没有部署该服务，仍然是可以通过 overlay 跨主机网络进行调用的。

同时我们查看管理节点的 8081 是否有被监听：

```bash
$ lsof -i:8081
```

![docker swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/d_network8.png)

发现 go-gin-demo 虽然没有部署到管理节点上，但其端口在其他节点上面依然被监听着，**所以我们得出，整个 overlay 网络中，每个服务都可以通过任意一台集群内服务器访问。**

这里需要注意一下，服务器防火墙需要开通 docker 相关的端口，这里为了方便，就把服务器的防火墙关闭了：

```bash
$ systemctl stop firewalld.service # centos 7 关闭防火墙
```

- 部署 go-gin-demo 到两个节点上，访问其中一台服务器，验证 swarm 集群是否具备负载均衡：

```bash
$ docker service scale go-gin-demo=2
```
![docker swarm](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/d_network7.png)

这时我们随意访问一台服务器，多访问几次，会出现返回来的是另一台服务器的地址，说明 swarm 集群具备负载均衡的特性。
