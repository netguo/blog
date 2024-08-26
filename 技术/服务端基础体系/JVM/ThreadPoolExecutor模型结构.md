## Java中的线程池    

### 一、线程池的意义
我们有两种常见的创建线程的方法，一种是继承Thread类，一种是实现Runnable的接口，Thread类其实也是实现了Runnable接口。但是我们创建这两种线程在运行结束后都会被虚拟机销毁，如果线程数量多的话，频繁的创建和销毁线程会大大浪费时间和效率，更重要的是浪费内存。那么有没有一种方法能让线程运行完后不立即销毁，而是让线程重复使用，继续执行其他的任务哪？
这就是线程池的由来，很好的解决线程的重复利用，避免重复开销。


1. 线程是稀缺资源，使用线程池可以减少创建和销毁线程的次数，每个工作线程都可以重复使用。开销一是减少系统资源的消耗，其次保证了响应时间。
2. 可以根据系统的承受能力，调整线程池中工作线程的数量，防止因为消耗过多内存导致服务器崩溃。

### 二、线程池的实现
#### 2.1 线程池的模型
```
public interface Executor {
    void execute(Runnable command);
}

public interface ExecutorService extends Executor {

    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();
  
    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
public abstract class AbstractExecutorService implements ExecutorService {

    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }

    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
    ......
}
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
}
```
线程池模型，提供了两个接口，Executor只是简单定义了任务提交的execute方法。ExecutorService定义一些便利方法，以及线程池生命周期的管理。
抽象类AbstractExecutorService，实现了便利方法，以及任务创建。
ThreadPoolExecutor是jdk中给出的线程池实现模型。netty，guava等有一些差异化的线程池模型，jdk1.7以后也引入了ForkJoinPool线程池模型。

```
public class Executors {

    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }

    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    ......
 }

```
Executors是一个线程池工厂，构造了一些我们常用的线程池模型。

#### 2.2 ThreadPoolExecutor线程池模型实现
![](../picture/threadpoolcode.jpg)

##### 构造方法  
ThreadPoolExecutor的构造方法，我们看一下参数最全的构造函数  
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

        
##### execute方法
涉及到的内容较多，单独拉章节讨论
##### 三种队列
三种队列都实现了BlockingQueue接口，都是线程安全，队列的操作都是基于ReentrantLock，其中Linked是基于链表实现的，Arrary是基于数组实现的。
* 直接提交 SynchronousQueue。工作队列的默认选项是 SynchronousQueue，它将任务直接提交给线程而不保持它们。在此，如果不存在可用于立即运行任务的线程，则试图把任务加入队列将失败，因此会构造一个新的线程。此策略可以避免在处理可能具有内部依赖性的请求集时出现锁。直接提交通常要求无界 maximumPoolSizes 以避免拒绝新提交的任务。当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。  

* 无界队列 LinkedBlockingQueue。使用无界队列（例如，不具有预定义容量的 LinkedBlockingQueue）将导致在所有corePoolSize 线程都忙时新任务在队列中等待。这样，创建的线程就不会超过corePoolSize。（因此，maximumPoolSize的值也就无效了。）当每个任务完全独立于其他任务，即任务执行互不影响时，适合于使用无界队列；例如，在Web页服务器中。这种排队可用于处理瞬态突发请求，当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。  

* 有界队列 ArrayBlockingQueue。当使用有限的 maximumPoolSizes时，有界队列有助于防止资源耗尽，但是可能较难调整和控制。队列大小和最大池大小可能需要相互折衷：使用大型队列和小型池可以最大限度地降低CPU使用率、操作系统资源和上下文切换开销，但是可能导致人工降低吞吐量。如果任务频繁阻塞（例如，如果它们是 I/O边界），则系统可能为超过您许可的更多线程安排时间。使用小型队列通常要求较大的池大小，CPU使用率较高，但是可能遇到不可接受的调度开销，这样也会降低吞吐量。  

##### 拒绝策略  RejectedExecutionHandler  

```
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

    public static class AbortPolicy implements RejectedExecutionHandler {

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }

    public static class DiscardPolicy implements RejectedExecutionHandler {

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }

    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```
* CallerRunsPolicy：策略直接执行了该任务，没有使用线程池中线程来执行。线程调用运行该任务的execute本身。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。
* AbortPolicy（默认策略）：处理程序遭到拒绝将抛出运行时RejectedExecutionException，调用者自己去感知异常做相应的处理。
* DiscardPolicy：任务直接被丢弃
* DiscardOldestPolicy：如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重新执行该任务。

#### 2.3 executors中默认线程池
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

### 三、线程池的使用

#### 3.1 线程池的风险
死锁
资源依赖
请求过载

#### 3.2 线程池的参数设置  

jdk引入线程池这个概念，对系统带了诸多好处。但是有一点的是，对于线程池参数的设定并没有一个特别明确规范，我们看看常见的一些线程池参数设置方式。  
**核心线程数**

 方式 |  说明   
-|-
Nthreads = Ncpu x Ucpu x (1 + W/C)Ncpu = CPU核心数，Ucpu = CPU使用率，0~1，W/C = 等待时间与计算时间的比率| 该计算方式来自于Java并发实践。该方式，保证了充分利用了cpu的资源，在指定的cpu利用率的基础上，保证任务阻塞的过程中，充分利用cpu执行其它任务 |
线程数 = Ncpu /（1 - 阻塞系数）	| 该公式来自于，Java 虚拟机并发编程。跟第一公式其实是同一个方式。 |
计算密集型：coreSize，N+1；IO密集型：coreSize，2N| 如果是计算密集型，cpu充分利用，设置Ncpu即可，计算密集型的线程恰好在某时因为发生一个页错误或者因其他原因而暂停，刚好有一个“额外”的线程，可以确保在这种情况下CPU周期不会中断工作，cpu使用时间，阻塞时间不确定，假定是1:1得来就是2N


我们业务场景下，对于单个任务，cpu的等待时间明显大于cpu的使用时间的。但是具体的等待时间，只能根据历史的经验值来设定。其次我们也不可能让我们线程池占用cpu的太多使用率，毕竟我们java应用程序还有其它线程在执行。其次cpu高负荷运行，也是一种不健康的应用状态。

**阻塞队列**

对于jdk的线程池提供以下阻塞队列。对于我们业务场景来，一个异步调用需要实时返回。理论上我们需要保证在限定tps下，有足够的线程可以支持。但是看我们的应用场景，我们虽然是实时返回，但是有我们响应数据时间允许一定的冗余。当某个接口响应突然慢的时候，那么我们核心线程数可能会不够，配置一个合适的有界阻塞队列，可以保证任务都得以执行。并且对于队列中任务数量进行监控，便于及时的处理堵塞情况。

名称|描述
-|-
ArrayBlockingQueue|一个用数组实现的有界阻塞队列，此队列按照先进先出(FIFO)的原则对元素进行排序。支持公平锁和非公平锁。
LinkedBlockingQueue|一个由链表结构组成的有界队列，此队列按照先进先出(FIFO)的原则对元素进行排序。此队列的默认长度为Intege.MAX_VALUE，所以默认创建的该队列有容量危险。
PriorityBlockingQueue|一个支持线程优先级排序的无界队列，默认自然序进行排序，也可以自定义实现compareTo()方法来指定元素排序规则，不能保证同优先级元素的顺序。
DelayQueue|一个实现PriorityBlockingQueue实现延迟获取的无界队列，在创建元素时，可以指定多久才能从队列中获取当
前元素。只有延时期满后才能从队列中获取元素。
SynchronousQueue|同步队列，每一个put操作必须等待take操作，否则不能添加元素。支持公平锁和非公平锁。
SynchronousQueue的一个使用场景是在线程池里。Executors.newCachedThreadPool()就使用了
SynchronousQueue,这个线程池根据需要（新任务到来时）创建新的线程，如果有空闲线程则会重复使用，
线程空闲了 60秒后会被回收。
LinkedTransferQueue|一个由链表结构组成的无界阻塞队列，相当于其它队列，LinkedTransferQueue队列多了transfer和tryTransfer方法
LinkedBlockingDeque|一个由链表结构组成的双向阻塞队列。队列头部和尾部都可以添加和移除元素，多线程并发时，可以将锁的竞争最多降到一半。

**线程池分组**  

从我们业务场景上可以看到我们是在调用不同服务，而且实际应用场景，不同底层服务响应时间差异性较大。如果用同一个线程池，那么造成的影响是什么呢，如果有一个服务挂了，我们设置的超时时间过长，那么可能造成线程池里面大量的核心线程被占用。造成其它服务也不可用。解决方案是把线程池进行分组，实现不同的任务进行隔离。
我们调用服务响应时间的差异性，如果用同一个线程池对于核心线程池的数量也是比较难把控的。如果进行隔离，我们更方便配置出更合适的线程池数量。

**动态线程池**  

暂不考虑，实时调用，我们设计目的已经在尽量占用cpu资源。参数的变化，很难带来一些正面的收益。

