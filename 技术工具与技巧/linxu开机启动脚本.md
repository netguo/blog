---
layout: post
title: "linxu开机启动脚本"
date: 2015-10-26 21:32:45 +0800
comments: true
categories: Linux
---  
<!--more-->

##### 标签： chkconfig，rc.d  

### 开机启动脚本  
  
---  

#### 开机启动服务
linux开机启动时，首先会加载硬件信息和系统信息后，会加载需要启动的服务。首先查看/etc/inittab，查看服务启动的级别，在本机中看到如下，默认是3，启动3级别服务。  

```  
    # Default runlevel. The runlevels used are:
    #   0 - halt (Do NOT set initdefault to this)
    #   1 - Single user mode
    #   2 - Multiuser, without NFS (The same as 3, if you do not have networking)
    #   3 - Full multiuser mode
    #   4 - unused
    #   5 - X11
    #   6 - reboot (Do NOT set initdefault to this)
    #
    id:3:initdefault:
```     

然后在/etc/rc.d/rc3.d/  下可以看大runlevel3下启动的服务。例如：  

```  
lrwxrwxrwx. 1 root root 15 8月   1 06:20 K89rdisc -> ../init.d/rdisc
lrwxrwxrwx. 1 root root 17 8月   6 03:19 S01sysstat -> ../init.d/sysstat
lrwxrwxrwx. 1 root root 11 8月   1 06:20 S99local -> ../rc.local  
```  

可以看到该目录下的脚本都是以K和S开头的，K表示关机时关闭的服务，S表示开机时启动的服务。数字的话代表启动关闭顺序。同样可以看出服务都是软链接，软链接指向目录/init.d/，最后一个执行/rc.local。在rc.local脚本下可以执行一些本地脚本。  

##### 指定服务开机启动    

* 方法一    

在/rc.d/rc3.d/指定软链接，例如开启memcached      

`
ln -s /etc/init.d/sshd  /etc/rc.d/rc3.d/S90sshd  
`

* 方法二   

chkconfig添加   
man chkconfig看到如下，chkconfig 就是来变更runlevel信息的 
```
    chkconfig  -  updates  and queries runlevel information for system 
    services
```  

添加sshd：`chkconfig --add sshd`    
  
 ---  
  
#### 执行本地脚本   

自己本地写的一些脚本如果想在开机是运行  

##### rc.local    

在rc.d中讲过，在最后rc3.d中最后一个服务是指向rc.local脚本，在这个脚本中可以写一些执行本地脚本的shell  
例如想在开机时执行启动一个应用的脚本，添加一个shell即可。`./~/work/startup.sh`

##### 将脚本做成一个服务，设置开机启动服务    

脚本中必须有如下的配置：  

```
#  chkconfig: 345 100 100
#  description: app_start
case:$1
    start)  
    ...
    stop)  
    
    *)
    exit 1;  
esac          
```       

将脚本复制到/etc/init.d 下，添加服务开机启动即可 

