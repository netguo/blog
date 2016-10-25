---
layout: post
title: "docker-容器的通讯"
date: 2015-05-03 15:00:57 +0800
comments: true
categories: Linux 
---

<!--more-->  

##### 标签： docker , Linking Containers Together  

---  

* list element with functor item
{:toc}  

docker实现容器之间的通信主要有两种方式：制定一个容器端口与一个本地端口相对应，通过docker的--link方式
下面就这这种方式进行说明：

----------------------

###一 制定一个容器端口与一个本地端口相对应###

在docker运行一个容器时，docker run 可以通过-P/p命令暴露一个接口，并与本机制定的一个端口相对应，外部便可以通过本机端口去访问容器。详解如下：

----------------------

####P 命令####
    docker run -d -P training/webapp python app.py  

通过-P命令，可以为container自动分配一个高位的端口（49000~49900），本机通过5000端口，tcp协议，通过inspect和ps都可以查看到此时的端口状态，通过（inspect可以查看一个container的状态，会以json形式返回，通过 -f 制定具体的哪个结点，父子结点以.分割）


       "Ports": {
            "5000/tcp": [
                {
                    "HostIp": "0.0.0.0",
                    "HostPort": "49153"
                }
            ]
        }
     


----------


   
####p命令####
    docker run -d -p 127.0.0.1:5000:5000/udp training/webapp python app.py

1. 可以发现通过-p命令中有一个IP地址，两个端口号，一个通信协议，ip地址和第一个端口号是本地的IP地址和端口号      ，：之后的端口号指定的容器内部的一个端口号，后面还可以指定一个协议。  
2. 本地ip地址默认为（0.0.0.0），本地端口号默认为（5000），协议默认为（tcp）  
3. -p命令也可以同时制定多个端口。  
```
docker run -d -p 5000:5000  -p 18080:8080 training/webapp python app.py
```  
4. 通过`docker port containerid/name`命令可以查看容器端口，对应本地网络信息。  
```
docker port fbc70f396b40 5001  
0.0.0.0:5000
```

----------


###二  docker的--link方式###
-----------------

在学习--link之前，首先了解docker的命名，docker在运行一个容器时，如果不指定名字，docker会给其自动分配一个名字，也可以通过--name为每个容器指定一个具有自身含义名字，这个名字是唯一的，可以在docker之间相互引用。

####通过links进行通信####
Link实现了containers之间的可见性（discover each other），并交互信息。每当建立一个link，相当于在信息源容器和信息接受容器之间建立了一个管道。link的建立是通过docker 的--link参数实现的。

####实例：####  

**1.首先，**创建一个包含postgresql数据库的容器

    $ docker run -d --name db training/postgres

**2.然后，**建立一个web容器，并通过--link指定db容器。

 ① 参数：--link name/id:alias，可以为信息源容器指定一个别名，后续操作也可以通过这个别名来访问信息源容器。

    $ docker run -d -P --name web --link db:webdb training/webapp python app.py

 ② 此时也可以通过inspect查看web容器的link信息**

    $ docker inspect -f "{{ .HostConfig.Links }}" web
    [/db:/web/webdb]

**3.最后，**测试连接性  

 通过nsenter进入容器: 

    $sudo docker-pid web 
     24499
    $ sudo nsenter --target 24499 --mount --uts --ipc --net --pid
  

  在web容器中ping db容器。（可以直接通过别名进行操作）

    root@0a4642b90eb1:/# apt-get install -yqq inetutils-ping
    root@0a4642b90eb1:/# ping webdb 
    PING webdb (172.17.0.22): 48 data bytes
    56 bytes from 172.17.0.22: icmp_seq=0 ttl=64 time=0.161 ms
    56 bytes from 172.17.0.22: icmp_seq=1 ttl=64 time=0.158 ms
    56 bytes from 172.17.0.22: icmp_seq=2 ttl=64 time=0.159 ms
    56 bytes from 172.17.0.22: icmp_seq=3 ttl=64 time=0.171 ms
    56 bytes from 172.17.0.22: icmp_seq=4 ttl=64 time=0.171 ms
    56 bytes from 172.17.0.22: icmp_seq=5 ttl=64 time=0.166 ms



####--link信息的查询####
在container中，link信息主要存储在两个地方：环境变量和/etc/hosts中  

**1.环境变量方式**
当我们通过--link db:webdb方式链接一个容器时，docker会在容器的建立相应的环境变量。
实例：

    $docker run --rm --name web2 --link db:db training/webapp env
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    HOSTNAME=eda306e68078
    DB_PORT=tcp://172.17.0.44:5432
    DB_PORT_5432_TCP=tcp://172.17.0.44:5432
    DB_PORT_5432_TCP_ADDR=172.17.0.44
    DB_PORT_5432_TCP_PORT=5432
    DB_PORT_5432_TCP_PROTO=tcp
    DB_NAME=/web2/db
    DB_ENV_PG_VERSION=9.3
    HOME=/

通过env命令，便可以查看db的链接信息。可以通过这些环境变量,来配置应用需要连接db的网络信息。这些连接将会是安全和私有的。仅仅recipient container可以连接到source container。  
**2.  /etc/hosts配置文件**

  查看web的/etc/hosts，可以看到web自身已经webdb的网络信息都记录在/etc/hosts文件中。

    $sudo nsenter --target 30990 --mount --uts --ipc --net --pid
    root@c0aa68e6e299:/# cat /etc/hosts
    172.17.0.39	c0aa68e6e299
    127.0.0.1	localhost
    ::1	localhost ip6-localhost ip6-loopback
    fe00::0	ip6-localnet
    ff00::0	ip6-mcastprefix
    ff02::1	ip6-allnodes
    ff02::2	ip6-allrouters
    172.17.0.44	webdb

  重新启动db查看web的/etc/hosts配置信息

    $ docker restart db
    $ sudo nsenter --target 30990 --mount --uts --ipc --net --pid
    root@c0aa68e6e299:/# cat /etc/hosts
    172.17.0.39	c0aa68e6e299
    127.0.0.1	localhost
    ::1	localhost ip6-localhost ip6-loopback
    fe00::0	ip6-localnet
    ff00::0	ip6-mcastprefix
    ff02::1	ip6-allnodes
    ff02::2	ip6-allrouters
    172.17.0.48	webdb

可以看到web容器中/etc/hosts中webdb地址也发生相应的变化。


----------


###三 两者之间的比较
相对于-p/-P的方式，--link方式是基于docker内部实现的，不会暴露端口给外部，具有更好的安全性。
  
