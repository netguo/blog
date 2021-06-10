### 背景
shutdown这个概念，从操作系统，到线程都存在，之前从未仔细考虑过这个问题。最近遇到tomcat，kafka consumer，Java executorservice的shutdown的线上问题，特此做一下学习。  

### 概念  
graceful shutdown，按照我的理解来说，就是告诉系统，进程或者线程你工作的时间到了，把手头上的事情做完，就停止。某些工作耗时比较久可以保存一个备份。  

### 几种不同shutdown分析  

### _Thread_  
Java中一个线程的生命周期，从初始化到结束。JDK中并没有提供安全的终止一个线程的方式，JDK提供的中断机制，实际上一个协作机制，并不会去中断线程。
在Thread中
```  

    //会清除中断标识
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }    

    //判断是否终止
    public boolean isInterrupted() {
        return isInterrupted(false);
    }   

    private native boolean isInterrupted(boolean ClearInterrupted);

    // 中断线程
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
```



### _线程池_    

线程池提供两种shutdown的方法
shutdown() 
当线程池调用该方法时,线程池的状态则立刻变成SHUTDOWN状态。此时，则不能再往线程池中添加任何任务，否则将会抛出RejectedExecutionException异常。但是，此时线程池不会立刻退出，直到线程池中的任务都已经处理完成，才会退出。   


```  

```

shutdownNow() 
根据JDK文档描述，大致意思是：执行该方法，线程池的状态立刻变成STOP状态，并试图停止所有正在执行的线程，不再处理还在池队列中等待的任务，当然，它会返回那些未执行的任务。它试图终止线程的方法是通过调用Thread.interrupt()方法来实现的，这种方法的作用有限，如果线程中没有sleep 、wait、Condition、定时锁等应用,interrupt()方法是无法中断当前的线程的。所以，shutdownNow不代表线程池就一定立即就能退出，它可能必须要等待所有正在执行的任务都执行完成了才能退出   


### _tomcat_  
  
一个应用容器，如果没有一个优雅的shutdown过程会造成很多问题，一个web应用必定会跟一些外部服务打交道，如果没有正常关闭，可能会造成一些不必要的错误。当然肯定也会造成内部任务的丢失。  
tomcat有内置的ShutdownHook，来实现关闭时终止  
shutdown hook is simply an initialized but unstarted thread. When the virtual machine begins its shutdown sequence it will start all registered shutdown hooks in some unspecified order and let them run concurrently.