### 三板斧Java定位CPU使用高问题

1. TOP命令，查询消耗CPU高的进程号 PID，并记录下来，按下键盘"H"键，记录高消耗线程号，并将改线程号转换为十六进制

2. 使用 jstack [pid]  > xx.log 命令打印进程信息，为了定位准确，可以多来几次

3. 打开日志文件，找到十六进制的线程信息，可定位到具体类的某一行。

 

####  示例：

##### 1、查询消耗CPU高的进程号 PID，并记录下来

```
 top

top - 18:45:29 up 14 days, 23:27,  6 users,  load average: 3.18, 3.08, 2.64

Tasks: 299 total,   1 running, 297 sleeping,   0 stopped,   1 zombie

Cpu(s): 25.7%us,  1.2%sy,  0.3%ni, 72.6%id,  0.1%wa,  0.1%hi,  0.1%si,  0.0%st

Mem:     23641M total,    23388M used,      252M free,      261M buffers

Swap:    24583M total,        0M used,    24583M free,    12252M cached

 

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND

16511 icore     20   0 3075m 657m  17m S  201  2.8   2859:27 java

 8369 root      20   0  207m 7556 2892 S    3  0.0 798:30.19 runHpiAlarm

22123 load      21   1 92800  19m 8332 S    3  0.1   0:59.40 mdrv

最耗CPU的进程号16511

 

 按下键盘"H"键，记录高消耗线程号，并将改线程号转换为十六进制

top - 18:46:25 up 14 days, 23:28,  6 users,  load average: 3.10, 3.06, 2.66

Tasks: 2722 total,   4 running, 2717 sleeping,   0 stopped,   1 zombie

Cpu(s): 26.0%us,  1.4%sy,  0.4%ni, 71.4%id,  0.8%wa,  0.0%hi,  0.0%si,  0.0%st

Mem:     23641M total,    23395M used,      245M free,      261M buffers

Swap:    24583M total,        0M used,    24583M free,    12256M cached

 

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND

19691 icore     20   0 3075m 657m  17m R  100  2.8   1419:59 java

19690 icore     20   0 3075m 657m  17m R  100  2.8   1419:58 java

 8370 root      20   0  207m 7556 2892 S    2  0.0 497:56.23 runHpiAlarm

 8408 root      20   0  207m 7556 2892 S    1  0.0 299:23.67 runHpiAlarm

 

线程号：19691 、19690 转换为十六进制为：0x4ceb 、0x4cea
```
 

#### 2、使用 jstack [pid]  > xx.log 命令打印进程信息  

```
#jstack 16511 > 1.log

#jstack 16511 > 2.log

#jstack 16511 > 3.log
```
 

#### 3、打开日志文件，找到两个线程信息，如下
在此时，通常一个大的应用会有比较多的线程，可以通过grep一下，一般死锁都发生在我们应用内部线程中，例如grep包前缀。
```
"Thread-77" prio=10 tid=0x00007f58d4041800 nid=0x4ceb runnable [0x00007f58d175f000]
   java.lang.Thread.State: RUNNABLE
 at com.huawei.iiss.upadapter.common.oprindex.thread.OprUserIndexThread.run(OprUserIndexThread.java:61)
 at java.lang.Thread.run(Thread.java:662)

"Thread-76" prio=10 tid=0x00007f58d4a43800 nid=0x4cea runnable [0x00007f58bd066000]
   java.lang.Thread.State: RUNNABLE
 at com.huawei.iiss.upadapter.common.oprlog.thread.SaveOprLogThread.run(SaveOprLogThread.java:80)
 at java.lang.Thread.run(Thread.java:662)
```
 

找到以上红色信息，已经定位到JAVA具体代码，产看代码，发现死循环。。。速度改之

 

附：TOP命令中需要关注的值：
（1）load average：此值反映了任务队列的平均长度；如果此值超过了CPU数量，则表示当前CPU数量不足以处理任务，负载过高
（2）%us：用户CPU时间百分比；如果此值过高，可能是代码中存在死循环、或是频繁GC等
（3）%sy：系统CPU时间百分比；如果此值过高，可能是系统线程竞争激烈，上下文切换过多，应当减少线程数
（4）%wa：等待输入、输出CPU时间百分比；如果此值过高，说明系统IO速度过慢，CPU大部分时间都在等待IO完成
（5）%hi：硬件中断CPU百分比；当硬件中断发生时，CPU会优先去处理硬件中断；比如，网卡接收数据会产生硬件中断
（6）swap used：被使用的swap；此值过高代表系统因为内存不足在进行频繁的换入、换出操作，这样会影响效率，应增大内存量
（7）%CPU：进程使用CPU的百分比；此值高表示CPU在进行无阻塞运算等
