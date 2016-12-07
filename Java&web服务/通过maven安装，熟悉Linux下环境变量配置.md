### mvn下载    
在mvn官网下载自己需要的版本，并解压，安装。
http://maven.apache.org/download.cgi  

### 环境变量   

##### 首先要明白为什么要配置环境变量    

安装应用，配置环境变量是要配置PATH变量，应用中我们要执行一些命令，例如maven中的mvn，bash如何识别mvn呢，就是通过PATH中配置的路径来寻找的。

##### 其次什么是环境变量 
在Linux系统中，我们可以直接赋值变量，利用=的形式，自定义变量和环境变量的区别在于，环境变量在子进程中可以适用。例如一个档案中定义的变量，其它的档案也可以取得该值。把一个自定义变量变为环境变量通过export命令即可。`export PATH`  

 
### 环境的配置    


Linux下环境配置，有两种配置方式。  

##### 通过export配置  
刚刚在上述中有讲过，通过export命令设定为环境变量。也可以通过export给PATH设置新的值：   
 
```  
export MAVEN_HOME=/Users/***/work/dev/tools/apache-maven-3.3.3  
export PATH=$MAVEN_HOME/bin:$PATH 
```  

##### 通过系统启动时加载的shell来执行环境变量的配置，系统启动加载文件如下：   

系统启动时会初始化/etc/bash.bashrc和/etc/profile，~/.bashrc和~/.profile文件。可以在这些文件中加入配置系统变量的shell。  

##### 系统级：      
/etc/profile和/etc/profile是系统级的。Linux用户的shell都有权使用这些环境变量。  
在文件中加入如下：  
```  
export MAVEN_HOME=/Users/***/work/dev/tools/apache-maven-3.3.3  
export PATH=$MAVEN_HOME/bin:$PATH 
```    

##### 用户级：  
修改~/.bashrc文件，如果用的是zsh，可以修改~/.zshrc。环境变量的权限控制到用户级别。  
在文件中加入如下：  
```  
export MAVEN_HOME=/Users/***/work/dev/tools/apache-maven-3.3.3  
export PATH=$MAVEN_HOME/bin:$PATH 
```    

### 其它    
   
##### 文件目录添加软连接  
配置maven目录时，上面方式是通过直接设置文件目录。如果maven中有不同的版本，那需要继续修改环境配置，可以直接给目录加一个软连接。  
在此，区分一下软连接和硬链接，软链接会以路径的形式存在，硬链接会以文件副本的形式存在。    
示例：
```  
ln -s /Users/***/work/dev/tools/apache-maven-3.3.3 /Users/***/link/apache-maven  
```  
    
##### maven 升级    

上面已经讲过，给目录添加软连接，升级只需要安装最近的版本，并将软链接重新指到最新的文件目录。   
 
```  
rm apache-maven  
ln -s /Users/***/work/dev/tools/apache-maven-newversion /Users/***/link/apache-maven   
```

