
#### digitalocean搭建    
+ 初次注册可以返10美元。
+ 购买时，可以通过paypal购买，初次通过杭州银行的借记卡付款不成功，后续通过招行的信用卡付款成功。  
+ 建立Droplets，选择了512M内存，20GB硬盘的服务器。 系统选择了64位的的centos6.5    

#### shadowsocks安装  
现在shadowsocks有多个版本，Python版，libev， nodejs版。Python方便一健式安装，采用了python版。    [github地址](https://github.com/shadowsocks/shadowsocks)    
安装过程如下：
  
*  安装python-setuptools    

```  
yum install -y python-setuptools    
```  

*  安装pip  
  
```  
easy_install pip  
```  
*  通过pip安装shadowsocks  
    
```  
pip install shadowsocks  
```    

####  shadowsocks配置  
启动shadowsocks主要有两种方式  
  
#####  命令行启动    

```
sudo ssserver -p 443 -k password -m aes-256-cfb --user nobody -d start 
```     
 +  -p 端口  
 +  -k 密码    
 + -m 加密协议，aes-256-cfb比较安全，建议采用
 + -d  daemon mode   
  
##### 通过配置文件启动    

+ 创建/etc/shadowsocks.json文件，内容如下：    

```  
{
    "server":"my_server_ip",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"mypassword",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false
}  
```    
+ 启动    
  
```    
ssserver -c /etc/shadowsocks.json -d start  
```    
+ 开机启动  
在/etc/rc.local中添加上述启动命令  

#### 优化  
速度优化有多种方式，可以参考：[shadowsocks速度优化](http://wuchong.me/blog/2015/02/02/shadowsocks-install-and-optimize/)  
自己服务器，内存只有512在升级Linux内核过程中，发生失败，后通过锐速方式优化速度。    

+  `uname -r `，确定自己的内核版本，在锐速官网，找到对应的锐动版本，则可以直接安装，如果不能，则先更换内核。  
+  安装    
  
```  
wget http://my.serverspeeder.com/d/ls/serverSpeederInstaller.tar.gz
tar xzvf serverSpeederInstaller.tar.gz
bash serverSpeederInstaller.sh  
```  
输入用户名密码，一路回车就好。  

+ 参数设定  

```  
rsc="1" #RSC网卡驱动模式  
advinacc="1" #流量方向加速  
maxmode="1" #最大传输模式  
gso="1" #支持gso算法  
```  

+ 重新启动锐速  `service serverSpeeder restart`


#### 其它  
+ shadowsocks日志路径：`/var/log/shadowsocks.log`  
+ iptables配置，如果存在，在etc/sysconfig/iptables加入配置：    

```  
-A INPUT -m state --state NEW -m tcp -p tcp --dport 8388 -j ACCEPT  
```



