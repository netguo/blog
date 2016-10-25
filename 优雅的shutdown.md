### 背景
shutdown这个概念，从操作系统，到线程都存在，之前从未仔细考虑过这个问题。最近遇到tomcat，kafka consumer，Java executorservice的shutdown的线上问题，特此做一下学习。  

### 概念  
graceful shutdown，按照我的理解来说，就是告诉系统，进程或者线程你工作的时间到了，把手头上的事情做完，就停止。某些工作耗时比较久可以保存一个备份。  

### 几种不同shutdown分析  

### _Thread_  
Java中一个线程的生命周期，从初始化到结束。JDK中并没有提供安全的终止一个线程的方式，JDK提供的中断机制，实际上一个协作
### _tomcat_  
  
一个应用容器，如果没有一个优雅的shutdown过程会造成很多问题，一个web应用必定会跟一些外部服务打交道，如果没有正常关闭，可能会造成一些不必要的错误。当然肯定也会造成内部任务的丢失。  
tomcat有内置的ShutdownHook，来实现关闭时终止  
shutdown hook is simply an initialized but unstarted thread. When the virtual machine begins its shutdown sequence it will start all registered shutdown hooks in some unspecified order and let them run concurrently.
