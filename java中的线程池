### Java中的线程池    

线程池ThreadPoolExecutor， 为了便于跨大量上下文使用，此类提供了很多可调整的参数和扩展钩子 (hook)。但是，建议使用较为方便的 Executors 工厂方法     

+  Executors.newCachedThreadPool()（无界线程池，可以进行自动线程回收）
+  Executors.newFixedThreadPool(int)（固定大小线程池）  
+  Executors.newSingleThreadExecutor()（单个后台线程）      

它们均为大多数使用场景预定义了设置。否则，在手动配置和调整此类时，参考ThreadPoolExecutor的构造函数。   
 
```  
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)    
  
```  
  
+ corePoolSize： 核心线程数，如果运行的线程少于 corePoolSize，则创建新线程来处理请求，即使其他辅助线程是空闲的。  
+ maximumPoolSize：最大线程数，如果运行的线程多于 corePoolSize 而少于 maximumPoolSize，则仅当队列满时才创建新线程。
+ keepAliveTime：线程存活时间  
+ unit：keepAliveTime的时间戳  
+ workQueue：工作队列，用来存放为执行的任务  
+ threadFactory：executor用来创建线程的线程工厂  
+ handler：当工作队列已满，拒绝新的线程的处理方法    

newCachedThreadPool()，newFixedThreadPool(int)，newSingleThreadExecutor()的实现都是通过TreadPoolExecutor的构造方法实现。

```  
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }  
      
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }    
    
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }    
    
```       
看Executor是通过execute来执行   

```  
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }  
```