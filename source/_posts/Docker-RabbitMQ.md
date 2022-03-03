---
title: 基于docker的rabbitmq集群
date: 2022-01-30 22:27:05
category:
- Docker
tags:
- RabbitMQ
---

### 架构简介

![架构简介](架构简介.png)

### 安装基于docker的rabbitmq

#### 安装rabbitMQ

```
docker run -d --hostname rabbit1 --name rabbit1 -p 5672:5672  -e RABBITMQ_ERLANG_COOKIE='rabbitmq' rabbitmq:3

docker run -d --hostname rabbit2 --name rabbit2 -p 5673:5672 --link rabbit1:rabbit1 -e RABBITMQ_ERLANG_COOKIE='rabbitmq' rabbitmq:3

docker run -d --hostname rabbit3 --name rabbit3 -p 5674:5672 --link rabbit1:rabbit1 --link rabbit2:rabbit2 -e RABBITMQ_ERLANG_COOKIE='rabbitmq' rabbitmq:3

注意RABBITMQ_ERLANG_COOKIE必须一样的,可以自定义.
```

#### 连接到rabbit1执行操作

```
docker exec -it rabbit1 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl start_app

此时的集群情况:
root@rabbit1:/# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit1 ...
[{nodes,[{disc,[rabbit@rabbit1]}]},
 {running_nodes,[rabbit@rabbit1]},
 {cluster_name,<<"rabbit@rabbit1">>},
 {partitions,[]},
 {alarms,[{rabbit@rabbit1,[]}]}]
```

#### 连接到rabbit2,将他加入到rabbit1的集群

```
docker exec -it rabbit2 bash
 rabbitmqctl stop_app
 rabbitmqctl reset
 rabbitmqctl join_cluster rabbit@rabbit1
 rabbitmqctl start_app
 此时可以查看集群的情况:
 rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2]}]},
 {running_nodes,[rabbit@rabbit1,rabbit@rabbit2]},
 {cluster_name,<<"rabbit@rabbit1">>},
{partitions,[]},
 {alarms,[{rabbit@rabbit1,[]},{rabbit@rabbit2,[]}]}]
```

#### 连接到rabbit3,将他加入到rabbit1的集群

```
docker exec -it rabbit3 bash
 rabbitmqctl stop_app
 rabbitmqctl reset
 rabbitmqctl join_cluster rabbit@rabbit1
 rabbitmqctl start_app
 此时可以查看集群的情况:
 rabbitmqctl cluster_status
 Cluster status of node rabbit@rabbit3 ...
[{nodes,[{disc,    [rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,    [rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]},
 {cluster_name,<<"rabbit@rabbit1">>},
 {partitions,[]},
 {alarms,[{rabbit@rabbit1,[]},{rabbit@rabbit2,[]},    {rabbit@rabbit3,[]}]}]    
```

#### 配置HAProxy,实现负载均衡

```
global
log 127.0.0.1 local0 info
maxconn 4096
daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    retries 3
    option redispatch
    maxconn 300
    timeout connect 5s
    timeout client 120s
    timeout server 120s

listen rabbitmq_local_cluster
    bind  127.0.0.1:5670
    mode tcp
    balance roundrobin
    server rabbit_1 127.0.0.1:5672 check inter 5000 rise 2 fall 3
    server rabbit_2 127.0.0.1:5673 check inter 5000 rise 2 fall 3
    server rabbit_3 127.0.0.1:5674 check inter 5000 rise 2 fall 3

listen private_monitoring
    bind 127.0.0.1:8100
    mode http
    option httplog
    stats enable
    stats uri /stats
    stats refresh 5s    
```

#### 启动haproxy

```
haproxy -f conf
```

#### 在浏览器输入http://localhost:8100/stats,查看结果.

![结果](结果.png)
