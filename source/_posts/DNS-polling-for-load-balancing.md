---
title: 负载均衡之dns轮询
date: 2022-01-29 22:52:23
category:
- DNS
tags:
- Load Balance
---

#### 简介

大多数域名注册商都支持对统一主机添加多条A记录，这就是DNS轮询，DNS服务器将解析请求按照A记录的顺序，随机分配到不同的IP上，这样就完成了简单的负载均衡。下图的例子是：有3台联通服务器、3台电信服务器，要实现“联通用户流量分摊到3台联通服务器、其他用户流量分摊到电信服务器”这个效果的设置。

![域名解析配置](域名解析配置.png)

DNS由于成本较低，所以一般在小型的网站用的比较多。但是大型的网站一般也会将用它和其他负载均衡的方式结合起来一起使用，DNS轮询方式提供的IP地址，在大型网站中往往是一个集群的地址，可能是均衡交换机也可能是均衡服务器。对于小网站的话，挂接多台服务器也没有问题。如：

![nslookup百度](nslookup百度.png)

#### DNS轮询的优点

1. 零成本：只是在DNS服务器上绑定几个A记录，域名注册商一般都免费提供解析服务；
2. 部署简单：就是在网络拓扑进行设备扩增，然后在DNS服务器上添加记录。

#### DNS轮询的缺点

- 可靠性低

假设一个域名DNS轮询多台服务器，如果其中的一台服务器发生故障，那么所有的访问该服务器的请求将不会有所回应，这是任何人都不愿意看到的。即使从DNS中去掉该服务器的IP，但在Internet上，各地区电信、网通等宽带接入商将众多的DNS存放在缓存中，以节省访问时间，DNS记录全部生效需要几个小时，甚至更久。所以，尽管DNS轮询在一定程度上解决了负载均衡问题，但是却存在可靠性不高的缺点。

- 负载分配不均匀

DNS负载均衡采用的是简单的轮询算法，不能区分服务器的差异，不能反映服务器的当前运行状态，不能做到为性能较好的服务器多分配请求，甚至会出现客户请求集中在某一台服务器上的情况。

DNS服务器是按照一定的层次结构组织的，本地DNS服务器会缓存已解析的域名到IP地址的映射，这会导致使用该DNS服务器的用户在一段时间内访问的是同一台Web服务器，导致Web服务器间的负载不均匀。此外，用户本地计算机也会缓存已解析的域名到IP地址的映射。当多个用户计算机都缓存了某个域名到IP地址的映射时，而这些用户又继续访问该域名下的网页，这时也会导致不同Web服务器间的负载分配不均匀。

负载不均匀可能导致的后果有：某几台服务器负荷很低，而另几台服务器负载很高、处理缓慢；配置高的服务器分配到的请求少，而配置低的服务器分配到的请求多。

#### 可靠性底的解决方案

- 架构图

![架构图](架构图.png)

- 具体步骤

  实现域名的解析,获取域名所有的A记录解析IP列表,对IP列表进行HTTP级别的探测

- 代码实现

```python
[dns_loop.py] []#!/usr/bin/env python
import dns.resolver
import os
import httplib

iplist = []
appdomain="admin888.yinmimoney.com"


def get_iplist(domain=""):
    try:
        A = dns.resolver.query(domain,'A')
    except Exception,e:
        print "dns resolver error:"+str(e)
        return
    for i in A.response.answer:
        for j in i.items:
            iplist.append(j.address)
    return True

def checkip(ip):
    checkurl = ip+":80"
    getcontent = ""
    httplib.socket.setdefaulttimeout(5)
    conn = httplib.HTTPConnection(checkurl)

    try:
        conn.request("GET","/admin/login",headers = {"Host":appdomain})
        r = conn.getresponse()
        getcontent = r.read(15)
    except Exception,e:
        print "checkip "+ip+" error:"+str(e)
    finally:
        if getcontent == "<!DOCTYPE html>":
            print ip + " [OK]"
        else:
            print ip + " [Error] "+ getcontent

if  __name__=="__main__":
    if get_iplist(appdomain) and len(iplist)>0:
        for ip in iplist:
            checkip(ip)
    else:
        print "dns resolver error."
```
