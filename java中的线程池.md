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
+ keepAliveTime：线程存活时间，一般指大于corePoolSize，少于maximumPoolSize的线程  
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
#### 三种队列  
* 直接提交 SynchronousQueue。工作队列的默认选项是 SynchronousQueue，它将任务直接提交给线程而不保持它们。在此，如果不存在可用于立即运行任务的线程，则试图把任务加入队列将失败，因此会构造一个新的线程。此策略可以避免在处理可能具有内部依赖性的请求集时出现锁。直接提交通常要求无界 maximumPoolSizes 以避免拒绝新提交的任务。当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。  

* 无界队列 LinkedBlockingQueue。使用无界队列（例如，不具有预定义容量的 LinkedBlockingQueue）将导致在所有corePoolSize 线程都忙时新任务在队列中等待。这样，创建的线程就不会超过corePoolSize。（因此，maximumPoolSize的值也就无效了。）当每个任务完全独立于其他任务，即任务执行互不影响时，适合于使用无界队列；例如，在Web页服务器中。这种排队可用于处理瞬态突发请求，当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。  

* 有界队列 ArrayBlockingQueue。当使用有限的 maximumPoolSizes时，有界队列有助于防止资源耗尽，但是可能较难调整和控制。队列大小和最大池大小可能需要相互折衷：使用大型队列和小型池可以最大限度地降低CPU使用率、操作系统资源和上下文切换开销，但是可能导致人工降低吞吐量。如果任务频繁阻塞（例如，如果它们是 I/O边界），则系统可能为超过您许可的更多线程安排时间。使用小型队列通常要求较大的池大小，CPU使用率较高，但是可能遇到不可接受的调度开销，这样也会降低吞吐量。  

#### 拒绝策略  RejectedExecutionHandler  
* CallerRunsPolicy：线程调用运行该任务的execute本身。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。池中已经没有任何资源了，那么就直接使用调用该execute的线程本身来执行。

* AbortPolicy：处理程序遭到拒绝将抛出运行时RejectedExecutionException

* DiscardPolicy：不能执行的任务将被删除

* DiscardOldestPolicy：如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序。

#### 线程池的大小
这个<<并发编程实战>>中给出的一些建议如下，具体线程池大小还是需要根据具体业务来处理  

* 对于计算密集的任务，在拥有 N 个处理器的系统上，当线程池大小为N+1时，通常能实现最优的利用率，即使当计算密集型的线程偶尔由于页缺失故障或者其它原因而暂停时，这个“额外”的线程也能确保CPU的时钟周期不会被浪费，对于包含IO操作或者其它阻塞操作的任务，由于线程并不会一直执行，因此线程池的规模应该更大。  

* 计算每个任务对该资源的需求量，然后用该资源的可用总量乘以每个任务的需求量，所得结果就是线程池大小的上限。 

* 当任务需要某种通过资源池来管理的资源时，例如数据库连接，那么线程池和资源池的大小将会相互影响。如果每个任务都需要一个数据库连接，那么连接池的大小就限制了线程池的大小。同样，当线程池中的任务是数据库连接的唯一使用者时，那么线程池的大小又将限制连接池的大小。