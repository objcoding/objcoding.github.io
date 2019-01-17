---
layout: post
title: "搭建Redis主从+哨兵模式"
categories: Redis
tags: sentinel
author: zch
---

* content
{:toc}
纯粹是为了记录搭建的过程.









## 前戏

- 下载:

```bash
$ wget http://download.redis.io/releases/redis-5.0.3.tar.gz
```



- 解压:

```bash
$ tar -zxvf redis-5.0.3.tar.gz
```



- 编译:

```bash
$ make install PREFIX=/opt/redis/redis-5.0.3
```



- 拷贝配置文件:

```bash
$ cp redis.conf /opt/redis/redis-5.0.3/bin
$ cp redis-sentinel.conf /opt/redis/redis-5.0.3/bin
```







## 配置

- 配置 redis.conf

```bash
# 这里需要配置内网地址,不要配置localhost, 不然只能单机自己玩
bind 内网地址 
# 进程后台运行, 这个必须的
daemonize yes
# 如果是从服务器, 那么这里需要配置主服务器的地址和端口
slaveof 主地址 主端口
```



- 配置 sentinel.conf


```bash
# 哨兵监听的端口
port 26379
# 进程后台运行, 这个必须的
daemonize yes
# 监视一个名为mymaster的主服务器,这个主服务器的 IP 地址为 172.17.4.57,端口号为 6379
# 后面那个2指的是将这个主服务器判断为失效至少需要1个Sentinel同意(只要同意Sentinel的数量不达标,自动故障迁移就不会执行)
sentinel monitor mymaster 172.17.4.57 6379 2
```



## 入戏

- 启动 redis

```bash
$ ./redis-server redis.conf
```

- 启动哨兵监控

```bash
$ ./redis-sentinel sentinel.conf
```

- 查看主从状态

```bash
$ ./redis-cli -h xxx.xxx.xx.x -p 6379
> info replication;

# 主服务器:
# Replication
role:master
connected_slaves:2
slave0:ip=172.17.4.58,port=6379,state=online,offset=20654866,lag=0
slave1:ip=172.17.4.59,port=6379,state=online,offset=20654852,lag=1
master_replid:cd484ba407267626276822a76c387aafc77c78c0
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:20655090
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:19606515
repl_backlog_histlen:1048576

# 从服务器
# Replication
role:slave
master_host:172.17.4.57
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:20675712
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:cd484ba407267626276822a76c387aafc77c78c0
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:20675712
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:19627137
repl_backlog_histlen:1048576
```



- 查看哨兵状态

```bash
$ ./redis-cli -h xxx.xxx.xx.x -p 26379 info sentinel

# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.17.4.57:6379,slaves=2,sentinels=3

```

