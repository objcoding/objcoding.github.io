---
layout: post
title: "Docker容器的日志处理"
categories: Docker
tags: log ELK 
author: zch
---

* content
{:toc}
Docker有很多的日志插件，默认使用 json-file，只有使用json-file时，sudo docker logs -f 才可以显示，输入以下命令查看docker日志插件：

```bash
$ sudo docker info | grep Logging
```













这里先说明一下，当容器运行时，docker会在宿主机上创建一个该容器相关的文件，然后将容器产生的日志转存到该文件下。docker logs -f 命令就会找到该文件内容并显示在终端上。

我们都知道docker logs -f会将所有对应的服务日志输出到终端，无论服务的部署在哪个节点上，那么我现在提出一个问题，是否每个节点对应的容器文件，都会保存该服务的完整日志备份，还是只保存该节点服务对应容器产生的日志？

因为这个问题涉及到每个节点如果都用filebeat监听宿主机的容器日志文件，那么如果每个节点的容器日志都是一个完整的备份，日志就会重复，如果只是保存该节点上容器的日志，就不会。

答案是只保留该节点上容器的日志，docker logs -f 命令只不过在overlay网络模型上走了一层协议，把在其它节点上的相同的容器日志汇聚起来。




默认使用docker的json-file，首先配置daemon：

```bash
$ sudo dockerd \
--log-driver=json-file \
--log-opt labels=servicename
```

启动容器需要添加如下参数：

```bash
$ sudo docker service update --label servicename=test
```

或者直接在docker-compose.yml中标记：

```yml
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
      replicas: 3
    labels:
      servicename: go-gin-demoxxxxxxx
    logging:
      options:
        labels: "servicename"

networks:
  overlay:

```

在每个节点安装filebeat，并且filebeat.yml配置如下：

```yaml
filebeat.prospectors:
- type: log
   paths:
   		# 容器的日志目录
      - /var/lib/docker/containers/*/*.log
      # 因为docker使用的log driver是json-file，因此采集到的日志格式是json格式，设置为true之后，filebeat会将日志进行json_decode处理
   json.keys_under_root: true
   tail_files: true
output.logstash:
  hosts: ["172.17.10.114:5044"]
```

在logstash.conf中配置索引：

```

output {
  elasticsearch {
    action => "index"
    hosts => ["172.17.10.114:9200"]
    # 获取日志label
    index => "%{attrs.servicename}-%{+YYYY.MM.dd}"
  }
}
```

Dockerfile文件需要将项目输出的日志打印到stdout和stderr中，不然json-file日志驱动不会收集到容器里面输出的日志，sudo docker logs -f就在终端显示不了容器日志了，在Dockerfile中需加入以下命令：

```bash
RUN ln -sf /dev/stdout /xx/xx.log \ # info
	&& ln -sf /dev/stderr /xx/xx.log # error
```

或者在在项目的log4j配置输出控制台：

```xml
<Appenders>
    <Console name="Console" target="SYSTEM_OUT">
        <PatternLayout pattern="[%d{DEFAULT}]%m"/>
    </Console>
</Appenders>
```



如果日志需要记录容器id名称和镜像名称，在运行容器时可以加入以下参数：

```
--log-opt tag="{{.ImageName}}/{{.Name}}/{{.ID}}"
```

![tag](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/docker_log_driver_tag.png)

最终，json-file日志插件将容器打印到控制台的日志生成到本地 `/var/lib/docker/containers/*/`目录中，格式如下：

```json
{
    "log":"[GIN-debug] [WARNING] Now Gin requires Go 1.6 or later and Go 1.7 will be required soon.",
    "stream":"stderr",
    "attrs":{
        "tag":"chenghuizhang/go-gin-demo:v3@sha256:e6c0419d64e5eda510056a38cfb803750e4ac2f0f4862d153f7c4501f576798b/mygo.2.jhqptjugfti2t4emf55sehamo/647eaa4b3913",
        "servicename":"test"
    },
    "time":"2019-01-29T10:08:59.780161908Z"
}
```



在logstash中格式化日志：

```json
filter {
 grok {
    patterns_dir => "/etc/logstash/conf.d/patterns"
    match => {"message" => "%{TIMESTAMP_ISO8601:time}%{SERVICENAME:attr.servicename}%{DOCKER_TAG:attr.tag}"}
}
```