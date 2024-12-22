## 基础概念
#### 什么是JUC
java.util.concurrent包
JUC出现的原因？为java语言提供多线程同步结构的支撑
JUC解决了什么问题？用于解决多线程同步问题，给java开发者提供便利的函数和功能、结构
JUC包含了哪些内容？原子类、锁类、工具类（线程同步结构、线程池等）

#### 如何实现锁

1. 如何表示锁状态：无锁？有锁？ boolean state
2. 如何保证多线程抢锁线程安全，CAS？ Unsafe compareAndSwapXXX ->C++->汇编->cpu指令 lock cmpxchg
3. 如果处理获取不到锁的线程
4. 如何释放锁

## atomic
AtomicInteger/AtomicLong/... ```for(;;){CAS操作}```
DoubleAdd/LongAdd，解决Atomic在高并发环境下的自旋瓶颈
> LongAdder的基本思路就是分散热点，将value值分散到一个数组中Cell[]，不同线程会命中到数组的不同槽中Cell，各个线程只对自己槽中的那个值进行CAS操作，这样热点就被分散了，冲突的概率就小很多。如果要获取真正的long值，只要将各个槽中的变量值累加返回。

> 从空间方面考虑，LongAdder其实是一种“空间换时间”的思想


> 在低竞争的并发环境下 AtomicInteger 的性能是要比 LongAdder 的性能好，而高竞争环境下 LongAdder 的性能比 AtomicInteger高，适合高并发下多写少读场景

## lock
#### ReentrantLock
![image.png](https://cdn.nlark.com/yuque/0/2022/png/8364057/1645932682024-e75d72ac-7f41-4991-b74d-d888202b6550.png#clientId=u4d28caf3-1127-4&from=paste&height=971&id=u9d322827&originHeight=1941&originWidth=4581&originalType=binary&ratio=1&size=1314416&status=done&style=none&taskId=u62bb0132-90a1-4b86-b2ec-ad7278b2b45&width=2290.5)
公平锁，非公平锁，新进入的线程是直接进入队列，还是先去抢占，抢占不到再加入队列。
默认非公平锁，效率更高。

#### ReentrantReadWriteLock
共享锁、排他锁
将原来的锁，分割为两把锁：读锁、写锁。适用于读多写少的场景，读锁可以并发，写锁与其他锁互斥。写写互斥、写读互斥、读读兼容。
实现方式跟ReentrantLock类似

- 一个statue高16为标识读锁，低16位标识写锁
- 写锁获取：如果有线程持有写锁，且非当前线程，加入阻塞队列，是当前线程重入

                    如果有线程持有读锁，加入阻塞队列

- 读锁获取：如果有线程持有写锁，且非当前线程，加入阻塞队列，是当前线程继续执行
-                如果没有持有写锁，继续执行（共享）
- 注意上述线程持有，表示有线程抢过，不管是阻塞队列中
- 释放锁：重入次数-1，读锁statue的高16位，读的线程数，重入由threadlocal存储；写锁由status存储
- 公平锁：(读写)获取锁之前，判断AQS的阻塞队列里是否有其他线程正在等待，如果有排队去
- 非公平锁：写直接抢，读锁判断队列中第一个是否为写线程，如果是排队，否则直接抢
- 锁降级，写锁释放之前把当前锁降级为读锁，当前线程来读不需要排队

#### LockSupport
LockSupport.park()，线程会被挂起。LockSupport.unpark()，线程会被唤醒
t1线程{LockSupport.park()}，t2线程{LockSupport.unpark(t1)}

VS wait notify：
* 因为wait()方法需要释放锁，所以必须在synchronized中使用，否则会抛出异常IllegalMonitorStateException
* notify()方法也必须在synchronized中使用，并且应该指定对象
* synchronized()、wait()、notify()对象必须一致，一个synchronized()代码块中只能有一个线程调用wait()或notify()
以上诸多限制，体现出了很多的不足，所以LockSupport的好处就体现出来了。
在JDK1.6中的java.util.concurrent的子包locks中引了LockSupport这个API，LockSupport是一个比较底层的工具类，用来创建锁和其他同步工具类的基本线程阻塞原语
#### AQS

1. Abstract : 因为它并不知道怎么上锁。模板方法设计模式即可，暴露出上锁逻辑
2. Queue：线程阻塞队列
3. Synchronizer：同步
4. CAS+state 完成多线程抢锁逻辑
5. Queue 完成抢不到锁的线程排队

实现多线程竞争的同步队列，采用模版方法，具体的上锁方式由具体的继承类自己实现。
采用park 堵塞线程，采用unpark唤醒下一个线程 

## 工具类
#### CountDownLatch
* A synchronization aid that allows one or more threads to wait until
* a set of operations being performed in other threads completes._
CountDown叫倒数，Latch是门栓的意思，一个或者多个线程等待，一系列其它线程的操作执行完成
CountDownLatch latch = new CountDownLatch(int count)
latch.countDown() 
latch.await
采用AQS实现

#### CyclicBartier
* A synchronization aid that allows a set of threads to all wait for
* each other to reach a common barrier point.  CyclicBarriers are
* useful in programs involving a fixed sized party of threads that
* must occasionally wait for each other. The barrier is called
* cyclic because it can be re-used after the waiting threads are released._
循环栅栏，几个线程等待一个共同的屏障。循环的意义，几个线程释放后，栅栏可以被重用
CyclicBarrier barrier = new CyclicBarrier(count)
barrier.await()
#### Phaser
阶段同步器
Phaser它就更像是结合了CountDownLatch和CyclicBarrier，翻译一下叫阶段
Phaser是按照不同的阶段来对线程进行执行，就是它本身是维护着一个阶段这样的一个成员变量，当前我是执行到那个阶段，是第0个，还是第1个阶段啊等等，每个阶段不同的时候这个线程都可以往前走，有的线程走到某个阶段就停了，有的线程一直会走到结束。你的程序中如果说用到分好几个阶段执行 ，而且有的人必须得几个人共同参与的一种情形的情况下可能会用到这个Phaser

#### Semaphore
限流，几个许个证，拿到许可证，执行，拿不到等待。  

、、、
Semaphore s = new Semaphore(permits);
s.acquire()
s.release();
Semaphore s = new Semaphore(permits,fair);
、、、

#### Exchanger
一对线程可以交换数据，threadA.exchange(sss),threadB.exchange(ccc);
#### ThreadLocal
Map存在每一个线程中
弱引用：threadlocal问题，key是弱引用
普通的ThreadLocal不会造成内存泄露，但是如果与线程池联合使用时，就可能会有这个问题。
虚引用：jvm开发者使用






