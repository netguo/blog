### 安装    

```  
curl -sLk http://ompc.oss.aliyuncs.com/greys/install.sh|sh​  
```    

### 启动  
 
```   
  ./greys <PID>[@IP:PORT]﻿​  
  如果本地，只需要PID获取PID，可以通过jps,ps -ef|grep java   
```
### 常用命令使用  
进入greys后，通过help命令，可以查看每个命令的详细用法。例如`help watch `  

#### trace  
可以追踪一个函数详细的每一步的消耗时间。
```  
参数说明

命令有以下参数

class-pattern：类名表达式匹配
method-pattern：方法名表达式匹配    

sample:  
trace *.PostLoanV2Impl monitorV2  

返回:  
Affect(class-cnt:1 , method-cnt:1) cost in 382 ms.
`---+Tracing for : thread_name="DubboServerHandler-192.168.6.76:20880-thread-8" thread_id=0xcb;is_daemon=true;priority=5;
    `---+[487,485ms]cn.fraudmetrix.creditcloud.api.impl.PostLoanV2Impl:monitorV2()
        +---[14,12ms]cn.fraudmetrix.creditcloud.object.request.PostLoanMonitorInfoV2:isValidate(@162)
        +---[14,0ms]cn.fraudmetrix.creditcloud.object.request.PostLoanMonitorInfoV2:getPartnerCode(@167)
       ....
```  
#### watch    
比较方便的观察，一个函数的入参，结果，异常，以及消耗的时间等等。
```  
+---------+----------------------------------------------------------------------------------+
|   USAGE | -[bfesx:En:] class-pattern method-pattern express condition-express              |
|         | Display the details of specified class and method                                |
+---------+----------------------------------------------------------------------------------+
| OPTIONS |              [b] | Watch before invocation                                       |
|         |              [f] | Watch after invocation                                        |
|         |              [e] | Watch after throw exception                                   |
|         |              [s] | Watch after successful invocation                             |
|         |             [x:] | Expand level of object (0 by default)                         |
|         |              [E] | Enable regular expression to match (wildcard matching by def  |
|         |                  | ault)                                                         |
|         |             [n:] | Threshold of execution times   
|         | -----------------+-------------------------------------------------------------- |                               |
|         |    class-pattern | Path and classname of Pattern Matching                        |
|         |   method-pattern | Method of Pattern Matching                                    |
|         |          express | express, write by OGNL.                                       |
|         |                  | FOR EXAMPLE    params[0]                                      |
|         |                  |     params[0]+params[1]                                       |
|         |                  |     returnObj,throwExp,target,clazz,method                                                                                           
|         |                  | THE STRUCTURE                                                 |
|         |                  |           target : the object                                 |
|         |                  |            clazz : the object's class                         |
|         |                  |           method : the constructor or method                  |
|         |                  |     params[0..n] : the parameters of method                   |
|         |                  |        returnObj : the returned object of method              |
|         |                  |         throwExp : the throw exception of method              |
|         |                  |         isReturn : the method ended by return                 |
|         |                  |          isThrow : the method ended by throwing exception     |
|         |  condition-expre | Conditional expression by OGNL                                |
|         |               ss |                                                               |
|         |                  | FOR EXAMPLE                                                   |
|         |                  |      TRUE : 1==1                                              |
|         |                  |      TRUE : true                                              |
|         |                  |     FALSE : false                                             |
|         |                  |      TRUE : params.length>=0                                  |
|         |                  |     FALSE : 1==2                                              |
|         |                  |                                                               |
|         |                  | THE STRUCTURE                                                 |
|         |                  |           target : the object                                 |
|         |                  |            clazz : the object's class                         |
|         |                  |           method : the constructor or method                  |
|         |                  |     params[0..n] : the parameters of method                   |
|         |                  |        returnObj : the returned object of method              |
|         |                  |         throwExp : the throw exception of method              |
|         |                  |         isReturn : the method ended by return                 |
|         |                  |          isThrow : the method ended by throwing exception     |
+---------+----------------------------------------------------------------------------------+
示例：  
1. 观察一个函数的输出结果
watch -f *.PostLoanV2Impl monitorV2  returnObj -x 1
-x 后面跟的是展开的层级  

2. 观察一个函数的入参（加入判断条件）  
watch -b *.PostLoanV2Impl monitorV2 params[0]  

3. 条件表达式  
watch -bE cn.fraudmetrix.creditcloud.api.impl.PostLoanV2Impl monitorV2  params[0]==null  

4. 观察表达式（可以获取一个对象的熟悉，一个map中key的值等。）
watch -b cn.fraudmetrix.creditcloud.api.impl.PostLoanV2Impl monitorV2  params[0].reportId  
注意，此处包路径，最好为绝对路径。
```
#### tt      

```    
时间隧道，记录一段时间一个函数的入参，返回值，抛出的异常，消耗时间等完整的过程。  

示例：  
tt -n 3 *.PostLoanV2Impl monitorV2  
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|    INDEX | PROCESS-ID |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1001 |       1001 |  2016-07-26 14:25:09 |          8 |     true |    false |      0x7b6457c1 |                 PostLoanV2Impl |                      monitorV2 |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1002 |       1002 |  2016-07-26 14:25:29 |          4 |     true |    false |      0x7b6457c1 |                 PostLoanV2Impl |                      monitorV2 |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1003 |       1003 |  2016-07-26 14:25:43 |          1 |     true |    false |      0x7b6457c1 |                 PostLoanV2Impl |                      monitorV2 |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
```
  
#### ptrace    

命令为trace命令的强化版，通过指定渲染路径来完成对方法执行路径的渲染过程。命令能主动搜索`tracing-path-pattern`所渲染的路径，渲染和统计整个调用链路上的所有性能开销和追踪调用的链路。    
  
```
参数说明：
class-pattern   类名表达式匹配
method-pattern  方法名表达式匹配
condition-express   条件表达式
express 观察表达式
[b] 在方法调用之前观察
[e] 在方法异常之后观察
[s] 在方法返回之后观察
[f] 在方法结束之后(正常返回和异常返回)观察  
  
示例：  
ptrace -n 5 cn.fraudmetrix.creditcloud.api.impl.PostLoanV2Impl monitorV2  params[0].reportId
```
### 其它功能    

#### 输出到文件  

```  
 ./greys.sh 4567|tee -a ./greys.log  
```  
#### sudo支持  
一般线上管理环境不会支持开放JVM部署用户权限，而是通过sudo-list来控制和监控用户的越权操作。由于greys.sh脚本中会对当前用户的环境变量产生感知，所以需要加上-H参数
```
sudo -u admin -H ./greys.sh 12345 
```  

#### 回话与任务  
Greys是一个C/S架构程序，所以当Client访问到Server时，Server会维护一个session，以及session的心跳、超时机制。事务(Tx)机制则是建立在session的基础上，所有的命令交互都会创建一个事务，并且产生对应的队列进行输出缓冲。
事务伴随着命令的生命周期而存在，命令分两种：
#### 立即返回  
立即返回的命令定义是：敲下命令后Server端立即返回最终结果，后续无持续反馈信息，释放Client对输入的锁定，重新开放让用户输入信息，比如version、sc、sm等  

#### 等待终止  
等待中止的命令则是需要用户主动输入Ctrl+D完成命令中止操作。命令执行后无法立即返回最终结果，而是不断的将中间产生的输出源源不断的输出到客户端中，如stack、monitor、trace

### 表达式  
表达式的引用能综合这两款软件的特长同时弥补他们的不足。目前Greys所采用的是OGNL表达式。  

### 不适合的场景   

#### 性能环境下的性能损耗定位
性能分析需要有更专业的软件，我自己常用的则是JProfiler（当然是付费的了）。Greys虽然性能损耗很小，但其分析的维度太少，所以只适合做简单的性能损耗定位。当你用过专业的性能分析软件之后，就会发现什么叫专业！

#### 开发环境下的远程DEBUG
虽然Greys能取代部分的远程DEBUG行为，但毕竟没不像DEBUG工具那样可以看到局部变量的值，而且可操作性上也没有JVM下Eclipse/IDEA等优秀的IDE自带的DEBUG工具这么人性化操作。

#### 线上环境大规模部署
与BTrace一样，Greys获取到的权限太高，如果线上大规模部署会遭受黑客的攻击，而今天我为了实现简单是没有做过多的鉴权控制。

#### JDK类库分析
JDK的类库存放在rt.jar中，启动时加载到BootstrapClassLoader中（Hotspot-JVM），但由于Greys也是用Java语言所编写，所以自身也用到了这些基础类库，默认情况下关闭了对这些类的增强。

当然，对于Spring、ibatis、Tomcat等三方类库是可以放心大胆使用的。

其它不适合场景

BTrace、HouseMD、Greys、JavOSize此类工具都会对Perm区、CodeCache（影响JIT）产生干扰，如果你的程序对这两块非常敏感，也请不要在这些场合下使用。