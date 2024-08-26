### 一、 概念
参考Thread的注释，可以大概总结为以下几点：
* 每一个线程有一个优先级，高优先级优先于低优先级执行。在学习Linux线程时，可以理解到获取时间片「相对」更多一些。
* 一个线程可以是守护线程，也可以不是守护线程。守护线程：用来服务于用户线程；不需要上层逻辑介入。例如GC线程。
* 一个线程创建一个线程时，会赋予子线程相同的优先级。如果父线程是守护线程，那么子线程也是守护线程。
* 虚拟机启动后，会启动一个main的非守护线程。
* Java虚拟机的退出条件
    * 调用System.exit方法，并且安全管理模块允许
    * 或者所有的非守护线程已经执行结束（正常返回或者抛出异常）

### 二、构造

### 三、核心字段和方法
#### 3.1 核心字段
比较容易理解，不做额外说明，主要理解thread的属性
```
    private volatile String name;
    
    /* Whether or not to single_step this thread. */
    private boolean     single_step;
    
    /* Whether or not the thread is a daemon thread. */
    private boolean     daemon = false;

    /* JVM state */
    private boolean     stillborn = false;
    /* What will be run. */
    private Runnable target;

    /* The group of this thread */
    private ThreadGroup group;
    
    /* The context ClassLoader for this thread */
    private ClassLoader contextClassLoader;

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    
    private volatile int threadStatus = 0;

    /**
     * The minimum priority that a thread can have.
     */
    public final static int MIN_PRIORITY = 1;

   /**
     * The default priority that is assigned to a thread.
     */
    public final static int NORM_PRIORITY = 5;

    /**
     * The maximum priority that a thread can have.
     */
    public final static int MAX_PRIORITY = 10;

```

####  3.2 核心方法
##### 1. yield
```
    //线程主动让出时间片
    public static native void yield();
    yeild的jvm实现较为简单，触发一次任务调度。
    static void do_sched_yield(void)
    {
        // ...
        current->sched_class->yield_task(rq);
        // ...
        schedule();
    }
```
通俗地讲，就是从当前CPU的运行队列中取出一个任务执行，并将前一个任务放回队列中去。具体调度可以从Linux的进程调度中可以理解到，普通进程的话就是CFS。
##### 2. sleep
```
    //sleep睡眠一段时间，到时间后重新调度
    public static native void sleep(long millis) throws InterruptedException;
    public static void sleep(long millis, int nanos) throws InterruptedException;

    //sleep的源码
    JVM_ENTRY(void, JVM_Sleep(JNIEnv* env, jclass threadClass, jlong millis))
  // ...
  if (millis == 0) {
    if (ConvertSleepToYield) { // 默认是false
      os::yield();
    } else {
      ThreadState old_state = thread->osthread()->get_state();
      thread->osthread()->set_state(SLEEPING);
      os::sleep(thread, MinSleepInterval, false); // 小睡一下
      thread->osthread()->set_state(old_state);
    }
  } else {
    ThreadState old_state = thread->osthread()->get_state();
    thread->osthread()->set_state(SLEEPING);
    if (os::sleep(thread, millis, true) == OS_INTRPT) {
      // 处理中断
    }
    thread->osthread()->set_state(old_state);
  }
  // ...
JVM_END

调用了os::sleep函数（JVM实现的os，并不是操作系统的sleep），linux平台的实现代码如下

int os::sleep(Thread* thread, jlong millis, bool interruptible) {
  ParkEvent * const slp = thread->_SleepEvent ;
  if (interruptible) {
    jlong prevtime = javaTimeNanos();

    for (;;) {
      if (os::is_interrupted(thread, true)) { 
        return OS_INTRPT;
      }

      jlong newtime = javaTimeNanos();

      if (newtime - prevtime < 0) {
        // ...
      } else {
        millis -= (newtime - prevtime) / NANOSECS_PER_MILLISEC;
      }
      
      if(millis <= 0) {
        return OS_OK;
      }
      // ...
      {
        // ...
        slp->park(millis);  // 调用的是os::PlatformEvent::park
        // ...
      }
    }
  } else {
    // ...
  }
}

最终是调用ParkEvent的park函数，实现如下

int os::PlatformEvent::park(jlong millis) {
  int v ;
  for (;;) {
      v = _Event ;
      if (Atomic::cmpxchg (v-1, &_Event, v) == v) break ; // cas设置_Event
  }
  if (v != 0) return OS_OK ;  // os::PlatformEvent::unpark的时候会设置_Event=1，这里就会提前跳出
  struct timespec abst;
  compute_abstime(&abst, millis);  // 0. 计算绝对时间

  int ret = OS_TIMEOUT;
  int status = pthread_mutex_lock(_mutex);  // 1. 加mutex锁
  // ...
  ++_nParked ;

  while (_Event < 0) {
    status = os::Linux::safe_cond_timedwait(_cond, _mutex, &abst); // 2. 等待
    if (status != 0 && WorkAroundNPTLTimedWaitHang) {
      pthread_cond_destroy (_cond);
      pthread_cond_init (_cond, os::Linux::condAttr()) ;
    }
    if (!FilterSpuriousWakeups) break ;                 // previous semantics
    if (status == ETIME || status == ETIMEDOUT) break ;
  }
  --_nParked ;
  if (_Event >= 0) {
     ret = OS_OK;
  }
  _Event = 0 ;
  status = pthread_mutex_unlock(_mutex); // 3. 释放mutex锁
  // ...
  return ret;
}
上面是一个典型的Mesa Monitor条件等待代码了，其中os::Linux::safe_cond_timedwait的代码比较简单，就是调用了pthread_cond_timedwait函数

int os::Linux::safe_cond_timedwait(pthread_cond_t *_cond, pthread_mutex_t *_mutex, const struct timespec *_abstime)
{
   if (is_NPTL()) {
      return pthread_cond_timedwait(_cond, _mutex, _abstime);
   } else {
      int fpu = get_fpu_control_word();
      int status = pthread_cond_timedwait(_cond, _mutex, _abstime);
      set_fpu_control_word(fpu);
      return status;
   }
}

````

通俗上来讲，sleep最终通过pthread_cond_timedwait实现了，线程等待。
pthread_cond_timedwait是Linux提供的线程等待函数，线程等待一定的时间，如果超时或有信号触发，线程唤醒。


##### join
join比较简单，主线程等待子线程执行完再执行
```
    //join
    public final synchronized void join(long millis) throws InterruptedException;
    public final synchronized void join(long millis, int nanos) throws InterruptedException;
    public final void join() throws InterruptedException;

    // join(long millis)的源码，可以看到根据isAlive判断子线程是否执行完成
    public final synchronized void join(long millis)throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0); //让出时间片，除非isAlive为false
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay); //让出时间片，且等待delay，除非isAlive为false或者delay小于0
                now = System.currentTimeMillis() - base;
            }
        }
    }
    

```
##### interrupt
Java中的中断，一定要跟操作系统的中断区分开来，Java中的中断是停止一个线程。
Thread.interrupt 的作用其实也不是中断线程，而是「通知线程应该中断了」，具体到底中断还是继续运行，应该由被通知的线程自己处理。

* 如果线程在wait,join,sleep返回一个中断异常InterruptedException，并且中断标记被清除。
* 如果等待一个可中断的InterruptibleChannel，那么InterruptibleChannel被关闭，并返回一个ClosedByInterruptException异常。并且线程的中断标志被设置为true。
* 如果被一个Selector阻塞，则直接返回选择操作。中断标志被设置为true.
* 正常情况：中断状态被设置为true，线程不会响应interrupt，需要自己手动处理，根据isInterrupted，自己处理中断后的处理操作。


``` 
    //判断是否中断，不清除中断标记
    public boolean isInterrupted(); 
    //判断是否中断，并且清除中断标记
    public static boolean interrupted();

    /* The object in which this thread is blocked in an interruptible I/O
     * operation, if any.  The blocker's interrupt method should be invoked
     * after setting this thread's interrupt status.
     */
    private volatile Interruptible blocker;
    //interrupt
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();  //非本线程，先权限检查

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);     // 先处理阻塞事件
                return;
            }
        }
        interrupt0();  // Just to set the interrupt flag
    }
```


### 参考：
[从源码层面解析yield、sleep、wait、park](https://juejin.im/post/6844903971463626766)  

[linux平台，对线程等待和唤醒操作的封装（pthread_cond_timedwait 用法详解）](http://javaquan.com/post/18850_1_1.html)