### 线上线下bug分析  
        
#### 列表：
  * 擅用IDE分析调试bug
  * 局域网远程调试bug
  * 线上bug调试神器Btrace  
  * javOSize
  * 线上日志分析   
    
#### 善用IDE分析调试bug   
ide提供了强大的bug调试功能，我们要善用，调试bug，往往打个断点并不能高效的调试出bug。之前从stackoverflow上看到过一个回答，摘抄了下来。    

* View the call stack at any point in time, giving you a context for your current stack frame.
* Step into libraries that you are not able to re-compile for the purposes of adding traces (assuming you have access to the debug symbols)
* Change variable values while the program is running
* Edit and continue - the ability to change code while it is running and immediately see the results of the change
* Be able to watch variables, seeing when they change
* Be able to skip or repeat sections of code, to see how the code will perform. This allows you to test out theoretical changes before making them.
* Examine memory contents in real-time
* Alert you when certain exceptions are thrown, even if they are handled by the application.
* Conditional breakpointing; stopping the application only in exceptional circumstances to allow you to analyse the stack and variables.
* View the thread context in multi-threaded applications, which can be difficult to achieve with tracing (as the traces from different threads will be interleaved in the output).
  
我用的ide是，intellij idea，可以方便的实现以上的线下调试功能。    

* 调用栈的输出，我们应用中的代码往往在栈底，框架中的代码在栈顶，但是异常抛出的第一行会解释什么异常。追踪异常，可以根据异常的解释，和具体到我们应用代码的那一行进行定位。  
* 动态改变程序的代码，可以通过热部署来实现。调试过程中，改变可以通过setValue来动态的改变运行中的程序的值。  
* 通过watch在intellij中可以比较方便的实现，add to watch ，intellij在程序中可以动态的显示已经执行过的值。  
* 在property上加断点可以定位在property值的变化。  
* 重复部分代码的执行（有待研究）  
* 实时查看内存的内容（有待研究）  
* 异常发生时，定位到该断点。（有待研究）  
* 条件断点，condition可以方便的配置。  
* 多线程应用中，调试不同的线程。  

#### 本地服务器远程调试  
tomcat remote  
ide都自带该功能，以intellij为例子：  
项目开放的远程端口为8090，new tomcat remote，填写远程地址，端口。  
并在Startup/Connection，Debug中address设置为8090，如下：  
```  
agentlib:jdwp=transport=dt_socket,address=8090,suspend=n,server=y  
```    

#### 线上追踪代码神器BTrace    
BTrace可以追踪线上数据调用情况。  
[API文档](https://btrace.kenai.com/javadoc/1.2/index.html)
Greys更为方便。可以参考Greys文档
  

#### 线上日志分析 
less      

看看less的定义：  
 Less is a program similar to more (1), but which allows backward  move-
       ment in the file as well as forward movement.  Also, less does not have
       to read the entire input file before  starting,  so  with  large  input
       files  it  starts  up  faster than text editors like vi (1).  Less uses
       termcap (or terminfo on some systems), so it can run on  a  variety  of
       terminals.   There is even limited support for hardcopy terminals.  (On
       a hardcopy terminal, lines which should be printed at the  top  of  the
       screen are prefixed with a caret  
主要功能与VIM相似，不过不能编辑，可以方便进行搜索，跳转首页，尾页，翻页等功能。
grep  
awk


#### 线上日志系统

随着公司的发展，现在已经有便利的日志系统，可以支持根据不同参数搜索，指定机器搜索，根据异常进行报警等功能。
