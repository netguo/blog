#### 初看execute
```  
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
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
当我们刚刚看到这个代码时，对于很多概念肯定是有些模糊的。例如ctl，addWork。甚至我们都没有从这段代码看到command是怎么执行的。我们先慢慢了解这些概念。

#### ctl
关于ctl，jdk给出了一大段文字给予解释
```
    /**
     * The main pool control state, ctl, is an atomic integer packing
     * two conceptual fields
     *   workerCount, indicating the effective number of threads
     *   runState,    indicating whether running, shutting down etc
     *
     * In order to pack them into one int, we limit workerCount to
     * (2^29)-1 (about 500 million) threads rather than (2^31)-1 (2
     * billion) otherwise representable. 
     ...
     */
     //默认为running，有效线程为0
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //29位，表示有效线程数量
    private static final int COUNT_BITS = Integer.SIZE - 3;（32）
    //容量2的29次方 -1 
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 线程池池运行状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    //有效线程数量
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    根据线程运行状态和有效线程数得到ctl值
    private static int ctlOf(int rs, int wc) { return rs | wc; }     
```
线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)，由于 int 型的变量是由32位二进制的数构成, 所以用 ctl 的高3位来表示线程池的运行状态, 用低29位来表示线程池内有效线程的数量. 由于这两部分信息在该类中很多地方都会使用到, 所以我们也经常会涉及到要获取其中一个信息的操作, 通常来说, 代表这两个信息的变量的名称直接用他们各自英文单词首字母的组合来表示, 所以, 表示线程池运行状态的变量通常命名为 rs, 表示线程池中有效线程数量的变量通常命名为 wc, 另外, ctl 也通常会简写作 c, 你一定要对这里提到的几个变量名稍微留个印象哦. 如果你在该类源码的某个地方遇到了见名却不知意的变量名时, 你在抱怨这糟糕的命名的时候, 要试着去核实一下, 那些变量是不是正是这里提到的几个信息哦.
线程池运行状态有
* RUNNING (运行状态): 能接受新提交的任务, 并且也能处理阻塞队列中的任务.
* SHUTDOWN (关闭状态): 不再接受新提交的任务, 但却可以继续处理阻塞队列中已保存的任务. 在线程池处于 RUNNING 状态时, 调用 shutdown()方法会使线程池进入到该状态. 当然, finalize() 方法在执行过程中或许也会隐式地进入该状态.
* STOP : 不能接受新提交的任务, 也不能处理阻塞队列中已保存的任务, 并且会中断正在处理中的任务. 在线程池处于 RUNNING 或 SHUTDOWN 状态时, 调用 shutdownNow() 方法会使线程池进入到该状态.
* TIDYING (清理状态): 所有的任务都已终止了, workerCount (有效线程数) 为0, 线程池进入该状态后会调用 terminated() 方法以让该线程池进入 
* TERMINATED 状态. 当线程池处于 SHUTDOWN 状态时, 如果此后线程池内没有线程了并且阻塞队列内也没有待执行的任务了 (即: 二者都为空), 线程池就会进入到该状态. 当线程池处于 STOP 状态时, 如果此后线程池内没有线程了, 线程池就会进入到该状态.
* TERMINATED : terminated() 方法执行完后就进入该状态.
#### workQueue
我们先从execute的实际使用入手，
#### 再看execute
addWork代码在后面，可以先看addWork代码，保证理解，需要我们注意的是&&逻辑（只要第一个条件不满足，后面条件就不再判断），这里会通过&&代表一些执行顺序。
```
   public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true)) //第一个参数是true代表用核心线程去启动
                return;
            addWork不成功，继续执行，获取ctl
            c = ctl.get();
        }
        //到此处表明核心线程已经不能运行任务
        //1. 线程池正在运行 2.把任务添加到队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //任务已经添加到队列，再次检查线程池状态，如果是非运行状态，把任务从队列中移除。拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 到这里线程池是运行状态
            // 查看有效线程池数量，如果是0的话，启动一个非核心线程，这个是为什么？？？
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }else if (!addWorker(command, false))  
        // 线程池非运行状态（这个addWork会直接返回false），或者队列添加任务失败(队列已满)
        // 所以这里是队列已经满时，启动一个非核心线程去运行，如果非核心线程不能运行则拒绝策略
            reject(command);
    }
    
```
可以看到，execute的方法，就如我们常见的线程池描述，1.如果有效线程数数小于核心线程数，去启动一个核心线程。2. 不能启动核心线程时，把任务加入队列。
3.如果加入队列失败，尝试去启动非核心线程。

#### addWorker
addWork主要作用是启动新的线程，去执行任务。
firstTask：就是实现runnable的任务
core：ture的话，有效线程数跟核心线程数比较，false的话有效线程数跟最大线程数比较。也就是

可以看到execute中核心逻辑在于addworker，我们先从addWorker入手，execute使用了三个参数：
addWorker(command, true) //线程数小于核心线程数时，启动一个新的线程
addWorker(null, false) // 启动一个非核心线程，任务置为空，从worker类，我们可以看到，是启动一个work线程，并且fisrtTask为空，会直接从队列中获取任务
addWorker(command, false)//启动一个非核心线程，并且首先执行command任务

```
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) { //无限循环
            int c = ctl.get();
            int rs = runStateOf(c);
            // 直接返回失败：：满足以下两个条件
            // 1. 线程池shutdown以上状态（STOP，TIDYING，TERMINATED），
            // 2. 排除（线程池为shutdown&&firstTask为null&&workQueque不为空），也就是说shutdown状态，firstTask为null时，队列不为空时可以继续往下走
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                //直接返回失败：：有效线程数大于有效线程的最大容量或者，大于核心线程or最大线程数根据core的布尔表达式
                // 这里我的理解是核心线程数设置的超级大，或者没有限制时，会考虑CAPACITY问题
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //有效线程数加一成功，跳出retry，跳出for循环：
                if (compareAndIncrementWorkerCount(c))  
                    break retry;  //可以看到此出是唯一可以跳出第一个for循环的成功场景。有效线程有新增。
                c = ctl.get();  // 如果线程池状态已经发生变化，重新走第一个for，重新校验线程池状态。
                if (runStateOf(c) != rs)
                    continue retry;
                // cas失败是因为有效线程数已经发生了变化，继续第二个循环
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        // 有效线程数增加一后。我们需要做的事情。
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask); //创建一个新的Work，并指明firstTask为当下任务
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    // 线程正在运行 或者 处于shutdown状态且提交的任务为空
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);  //添加到工作线程队列中（注意这个worker是工作线程队列）
                        int s = workers.size();
                        if (s > largestPoolSize) //largestPoolSize标识曾经有过的最大长度
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();  //该work的线程启动，从worker类中我们可以看到，线程启动实际上执行Work的run方法。
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

#### Worker
```
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        ....
```
先看Worker的模型结构，继承了AQS，并且实现了Runnable接口。
再看Worker的构造函数，firstTask指的构造当前work线程，第一个执行的任务，可以为null；启动一个新线程，并且以work为入参，也就是start后执行work的run方法。

#### runWorker
```
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
参考：https://blog.csdn.net/cleverGump/article/details/50688008

