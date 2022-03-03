---
title: 简单的docker应用栈
date: 2022-02-12 16:17:16
category:
- Docker
tags:
- 应用
---

### 架构

![架构](架构.png)

```
说明：这个应用栈包括6个节点，其中包括一个代理节点，两个web应用节点，一个主数据库节点，两个从数据节点。
```

### 实验环境说明

宿主机操作系统：centos7

容器[Docker](https://www.docker.com/)

代理和负载均衡[HAPoxy](http://www.haproxy.com/)

数据库[Radis](http://redis.io/)

编程语言[Python](https://www.python.org/)

应用框架[Django](https://www.djangoproject.com/)

### 准备

#### 获取应用栈各节点所需的镜像

```bash
docker pull django

docker pull haproxy

docker pull redis
```

#### 验证*docker*下载完情况：键入**docker images**

```bash
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
django              latest              2427265c139d        2 weeks ago         450.7 MB
redis               latest              fb46ec1d66e0        2 weeks ago         151.3 MB
haproxy             latest              4e2769f9caad        2 weeks ago         139.1 MB
```

### 应用栈容器节点启动

#### 启动Redis容器

```bash
docker run -it --name redis-master redis /bin/bash

docker run -it --name redis-slave1 --link redis-master:master redis /bin/bash

docker run -it --name redis-slave2 --link redis-master:master redis /bin/bash
```

#### 启动Django应用

```bash
docker run -it --name APP1 --link redis-master:db -v ~/Projects/Django/App1:/usr/src/app django /bin/bash

docker run -it --name APP2 --link redis-master:db -v ~/Projects/Django/App2:/usr/src/app django /bin/bash
```

#### 启动HAProxy容器

```bash
docker run  -it --name HAProxy --link APP1:APP1 --link APP2:APP2 -p 6301:6301 -v ~/Projects/HAProxy:/tmp haproxy /bin/bash
```

#### 检查容器运行情况，键入**docker ps**

```bash
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
91d571100cee        haproxy             "/bin/bash"              4 days ago          Up 2 days           0.0.0.0:6301->6301/tcp   HAProxy
894c4241f595        django              "/bin/bash"              4 days ago          Up 2 days                                    APP2
e4d6d2f394cb        django              "/bin/bash"              4 days ago          Up 2 days                                    APP1
84aa1adf394b        redis               "/entrypoint.sh /bin/"   6 days ago          Up 2 days           6379/tcp                 redis-slave2
f71b9c1874a0        redis               "/entrypoint.sh /bin/"   6 days ago          Up 2 days           6379/tcp                 redis-slave1
bf5d59eea503        redis               "/entrypoint.sh /bin/"   6 days ago          Up 2 days           6379/tcp                 redis-master
```

### 应用栈容器节点的配置

#### Redis Master主数据容器节点的配置

##### 配置redis master节点

1. 使用**docker inspect redis-master**命令查看容器所挂载*volume*的情况。在输出的内容中找到*Mounts*节点，中的*Source*的值就是*docker*机在宿主机的挂载点，形如：

   
```bash
   "Mounts": [
{
   "Name": "d8f67be0124d5f0bad604c60214b4b2e6bbb2296f8d08804ea1d2a1c7cce2b49",
   "Source": "/var/lib/docker/volumes/d8f67be0124d5f0bad604c60214b4b2e6bbb2296f8d08804ea1d2a1c7cce2b49/_data",
   "Destination": "/data",
   "Driver": "local",
   "Mode": "",
   "RW": true,
   "Propagation": ""
}
],
```

使用*cd*命令进入这个目录：

```bash
cd /var/lib/docker/volumes/d8f67be0124d5f0bad604c60214b4b2e6bbb2296f8d08804ea1d2a1c7cce2b49/_data 

vim redis.conf
```

输入

```bash
daemonize yes
pidfile /var/run/redis.pid
```

2. 进入redis-master容器，利用配置文件启动redis：


```bash
docker exec -it redis-master bash    
```

键入**ls**命令可以看到redis－master当前目录有了我们刚才编辑的redis.conf文件

```bash
cp redis.conf /usr/local/bin

cd /usr/local/bin

redis-server redis.conf
```

3. 同样的方法配置redis-slave*容器，只是这两个redis.conf文件稍微有点不一样。内容如下：

```bash
daemonize yes
pidfile /var/run/redis.pid
slaveof master 6379
```

##### Redis数据库容器节点测试


首先使用**docker exec**命令登录*redis-master*容器，启动*redis*客户端，并存储一个数据：

```bash
root@bf5d59eea503:/data# redis-cli
127.0.0.1:6379> set master helloworld
OK
127.0.0.1:6379> get master
"helloworld"
127.0.0.1:6379>
```

再使用**docker exec**命令登录*redis-slave*容器，启动*redis*客户端，获取存储数据：

 ```bash   
root@84aa1adf394b:/data# redis-cli
127.0.0.1:6379> get master
"helloworld"
127.0.0.1:6379>
```

证明redis数据库节点已经搭建好了


#### APP容器节点的配置

Django容器启动以后，需要利用Django框架，开发一个简单的Web程序。

1. 为了访问数据库，需要在容器中安装Python语言的Redis支持包，执行如下命令：

```bash
pip install redis
```

安装完后执行以下命令测试一下：

```bash
root@e4d6d2f394cb:/# python
Python 3.4.4 (default, Feb 17 2016, 02:50:56)
[GCC 4.9.2] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import redis
>>> print(redis.__file__)
/usr/local/lib/python3.4/site-packages/redis/__init__.py
>>>
```

看到类似这样的效果表示**python**的**redis**支持包安装好了

2. 利用docker exec命令登录到APP的/usr/src/app目录下,创建APP。执行以下命令：

```bash
cd /usr/src/app/
mkdir dockerweb
cd dockerweb
django-admin startproject redisweb
cd redisweb
python manage.py startapp helloworld
```

3. 在宿主机，做如下操作：

```bash
cd ~/Projects/Django/App1
cd dockerweb/redisweb/helloworld
vim views.py
```

输入以下内容：

```python
from django.shortcuts import render
from django.http import HttpResponse

import redis

def hello(request):
   str=redis.__file__
   str+="<br>"
   r=redis.Redis(host='db',port=6379,db=0)
   info=r.info()
   str+=("Set Hi <br>")
   r.set('Hi','HelloWorld-APP1')
   str+=("Get Hi: %s <br>" % r.get('Hi'))
   str+=("Redis info: <br>")
   str+=("Key: Info Value")
   for key in info:
         str+=("%s :%s <br>" % (key,info[key]))
   return HttpResponse(str)
```

4. 再执行以下命令：

```bash
cd ~/Projects/Django/App1/dockerweb/redisweb/redisweb
vim setting.py
```

在INSTALLED_APPS节点上添加helloworld项目，形如下：

```python
# Application definition

INSTALLED_APPS = [
      'django.contrib.admin',
      'django.contrib.auth',
      'django.contrib.contenttypes',
      'django.contrib.sessions',
      'django.contrib.messages',
      'django.contrib.staticfiles',
      'helloworld',
]
```

5. 再vim urls.py，修改urlpatterns节点，引入helloworld项目。形如下：

```python
from django.conf.urls import url
from django.contrib import admin
from helloworld.views import hello

urlpatterns = [
         url(r'^admin/', admin.site.urls),
         url(r'^helloworld$',hello)
]
```

6. 再次利用docker exec命令进入APP1容器中，执行如下命令：

``` bash
cd /usr/src/app/dockerweb/redisweb/
python manage.py makemigrations
python manage.py migrate
python manage.py runserver 0.0.0.0:8001
最后输出：
Performing system checks...

System check identified no issues (0 silenced).
March 02, 2016 - 16:24:08
Django version 1.9.2, using settings 'redisweb.settings'
Starting development server at http://0.0.0.0:8001/
Quit the server with CONTROL-C.
```


至此APP1应用启动完毕，APP2配置基本一样，只是修改**views.py**时，*hello*函数这句*r.set('Hi','HelloWorld-APP1')*改成**r.set('Hi','HelloWorld-APP2')**，最后启动应用时换一下端口，以备后面代理使用。命令*python manage.py runserver 0.0.0.0:8001*改成**python manage.py runserver 0.0.0.0:8002**


#### HAProxy容器节点的配置

1. 在宿主机上，执行如下过程：

```bash
cd ~/Projects/HAProxy
vim haproxy.cfg
```

键入如下内容：

```bash
global
log 127.0.0.1 local0 #日志输出配置，所有日志都记录在本机，通过local0输出
maxconn 4096 #最大连接数
chroot /usr/local/sbin #改变当前工作目录
daemon #以后台方式运行HAProxy
nbproc 4 #启动4个HAProxy实例
pidfile /usr/local/sbin/haproxy.pid #pid文件位置

defaults
      log 127.0.0.1 local3
      mode http #｛tcp｜http｜health｝设定启动实例的协议类型
      option dontlognull #保证HAProxy不记录上级负载均衡发送过来的用于检测状态没有数据的心跳包
      option redispatch #当serverId对应的服务器挂掉后，强制定向到其他的健康服务器
      retries 2 #重试两次连接失败就认为服务器不可用，主要通过后面的check检查
      maxconn 2000
      balance roundrobin #负载均衡算法，roundrobin表示轮询，source表示按照IP
      timeout connect 5000ms
      timeout client 50000ms
      timeout server 50000ms

listen redis_proxy
         bind  0.0.0.0:6301
      stats enable
      stats uri /haproxy-stats
      server APP1 APP1:8001 check inter 2000 rise 2 fall 5
      server APP2 APP2:8002 check inter 2000 rise 2 fall 5
```

1. 使用docker exec进入HAProxy容器中，执行以下过程：

```bash
cd /tmp/ 
cp haproxy.cfg /usr/local/sbin/
haproxy -f haproxy.cfg
```


至此完成所有的容器节点的配置！


### 应用栈访问测试

我的宿主机IP地址为172.16.89.129，故在浏览器上输入http://172.16.89.129:6301/helloworld即可看到效果。

**本文标题:**[简单的docker应用栈](https://ivivisoft.com/2016/03/02/a-docker-application-stack/)
