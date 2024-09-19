## 概述
容器主要分两种，Collection、Map。
从存储结构上来讲，分为两种，连续存储和非连续存储。
Collection主要分为三种：List、Set、Queue，Queue是后来加入的，主要为高并发使用，Queue实现了一个队列，并在基础之上实现了多线程访问方法（比如说put阻塞式放，get阻塞式的取），最重要的实现阻塞队列，初衷为了线程池、高并发做准备。

## HashTable
所有方法都加上synchronize

## HashMap
没有做任何同步处理
数组+链表，1.8之后hash碰撞后，链表超过阈值8后，转化为红黑树提高查找效率（On -> Ologn）
初始化数组大小，16，负载因子0.75，扩展采用左移一位=*2
每一个元素采用一个Entry{ key , value , next，hash}

## SynchronizedMap

```java
private final Map<K,V> m;     // Backing Map
final Object      mutex;        // Object on which to synchronize
public int size() {
    synchronized (mutex) {return m.size();}
}
```
如上所示，通过synchronized一个Object方法实现同步

## ConcurrentHashMap
采用分段锁技术，每段采用final Segment<K,V>[] segments，segment继承了ReentrantLock，具体的put，get操作在segment中实现，put加锁，中间数据采用了volatile保证数据可见性，get操作未加锁，获取具体的entry采用CAS。
1.8后舍弃了分段锁的实现方式，元素都存在Node数组中，每次锁住的都是一个Node对象，而不是某一段数组，所以支持写的并发度更高。再者它引入了红黑树，在hash冲突严重时，读操作效率更高。1.8扩容时，对rehash做了优化，扩容，槽数增加两倍，一个点要不在当前位置，要不在是0的话索引没变，是1的话索引变成“原索引+oldCap位置，一个标识为0、1，标志是否需要转移到原索引+oldCap位置，省去了重新hash的工作，是否在+oldCap位置通过高位与运算。

#### 1.7

**基本结构**

```java
ConcurrentHashMap{
	final Segment<K,V>[] segments;
	transient Set<K> keySet;
	transient Set<Map.Entry<K,V>> entrySet;
}
Segments{
       transient volatile HashEntry<K,V>[] table;
       transient int count;
       transient int modCount;
       transient int threshold;
       final float loadFactor;
}
HashEntry<K,V>{
	final int hash;
	final K key;
	volatile V value;
	volatile HashEntry<K,V> next;
	HashEntry(int hash, K key,V value,HashEntry)
}
```

**put**
```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //加锁！！！！！
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

**get**
```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    //采用CAS根据Hash获取元素
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```
#### 1.8
```java
public V put(K key, V value) {
        return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());//计算hash
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 数据为空，进行初始化，底下会介绍
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 找该 hash 值对应的数组下标，得到该下标处第一个节点 f
            // 此方法具有volatile语义
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果这个节点上是空的，直接构造新节点cas插入，cas失败说明其他线程插入了，继续for循环
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 发生了扩容导致hash==MOVED
            else if ((fh = f.hash) == MOVED)
                // 迁移节点，后边介绍
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) { //锁住头节点，别的线程就不能再操作该节点，即同一时间只能有一个线程对index处进行put
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 发现相等的key，判断是否进行覆盖然后break
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                // 没有相等key，去链表尾端新建节点
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 红黑树上插入新节点
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    // 判断是否转为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
}
```

1. 根据key计算出hashcode
2. 判断是否需要初始化，是的话进行初始化，初始化时while循环（table==null||table.size=0），有个volatile标志位，标识是否有线程正在初始化，如果有其它线程进行初始化，thread.yield()。
3. tabAt(hash)=null，cas头节点进行赋值，赋值成功返回，如果赋值失败重新循环
4. 如果进行扩容状态，迁移节点
5. 已经有头节点，synchronized锁定头节点进行插入

#### 为什么放弃分段锁

和hashmap一样，在jdk1.7中ConcurrentHashMap的底层数据结构是数组加链表
Segment继承了重入锁ReentrantLock，有了锁的功能，每个锁控制的是一段，当每个Segment越来越大时，锁的粒度就变得有些大了。
（1.7put对segment lock、segment不连续）

- 分段锁的优势在于保证在操作不同段 map 的时候可以并发执行，操作同段 map 的时候，进行锁的竞争和等待。这相对于直接对整个map同步synchronized是有优势的。
- 缺点在于分成很多段时会比较浪费内存空间(不连续，碎片化); 操作map时竞争同一个分段锁的概率非常小时，分段锁反而会造成更新等操作的长时间等待; 当某个段很大时，分段锁的性能会下降。

#### 为什么不用ReentrantLock而用synchronized ?

- 减少内存开销:如果使用ReentrantLock则需要节点继承AQS来获得同步支持，增加内存开销，而1.8中只有头节点需要进行同步。
- 内部优化:synchronized则是JVM直接支持的，JVM能够在运行时作出相应的优化措施：锁粗化、锁消除、锁自旋等等。

## ConcurrentSkipListMap

通过跳表实现高并发下的有排序容器，并发容器不采用tree因为效率太低，所以通过跳表+Map实现
跳表算法，的随机函数算法，可以在算法的时候学习

## Vector
我们来看最早的这个容器Vector，内部是自带锁的，你去读它的时候就会看到很多方法synchronized二话不说先加上锁在说，所以你用Vector的时候请放心它一定是线程安全的。

## List
ArrayList、LinkedList
CopyOnWriteArrayList 写时复制，是用写少读多的场景
SynchronizedList
```java
CopyOnWriteArrayList

private transient volatile Object[] array;
private E get(Object[] a, int index) {
    return (E) a[index];
}
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

## BlockingQueue

#### 概述

BlockingQueue，是我们后面讲线程池需要用到的这方面的内容，是给线程池来做准备的。BlockingQueue的概念重点是在Blocking上，Blocking阻塞，Queue队列，是阻塞队列。他提供了一系列的方法，我们可以在这些方法的基础之上做到让线程实现自动的阻塞。

#### 常用方法

boolean offer 添加数据返回一个boolean值
void add   添加数据失败抛出一个异常
boolean remove 
E pool  取数，并remove
E take  阻塞取，取不到线程阻塞
void put 阻塞放数据，添加不上，线程阻塞

#### LinkedBlockingQueue

LinkedBlockingQueue，体现Concurrent的这个点在哪里呢，我们来看这个LinkedBlockingQueue，用链表实现的BlockingQueue，是一个无界队列。就是它可以一直装到你内存满了为止，一直添加。

#### ArrayBlockingQueue

ArrayBlockingQueue是有界的，你可以指定它一个固定的值10，它容器就是10，那么当你往里面扔容器的时候，一旦他满了这个put方法就会阻塞住。然后你可以看看用add方法满了之后他会报异常。offer用返回值来判断到底加没加成功，offer还有另外一个写法你可以指定一个时间尝试着往里面加1秒钟，1秒钟之后如果加不进去它就返回了。

#### DelayQueue

DelayQueue可以实现在时间上的排序，这个DelayQueue能实现按照在里面等待的时间来进行排序。

#### SynchronousQueue

SynchronousQueue容量为0，就是这个东西它不是用来装内容的，SynchronousQueue是专门用来两个线程之间传内容的，给线程下达任务的。

#### TransferQueue

TransferQueue传递，实际上是前面这各种各样Queue的一个组合，它可以给线程来传递任务，以此同时不像是SynchronousQueue只能传递一个，TransferQueue做成列表可以传好多个。比较牛X的是它添加了一个方法叫transfer，如果我们用put就相当于一个线程来了往里一装它就走了。transfer就是装完在这等着，阻塞等有人把它取走我这个线程才回去干我自己的事情。

#### 整体理解

- 从Hashtable一直到这个ConcurrentHashMap，这些不是一个替代的关系，它们各自有各自的用途
- Vector到Queue的这样的一个过程，这里面经常问的面试题就是Queue到List的区别到底在哪里需要大家记住区别主要就是Queue添加了许多对线程友好的API offer、peek、poll，他的一个子类型叫BlockingQueue对线程友好的API又添加了put和take，这两个实现了阻塞操作。
- DelayQueue SynchronousQ TransferQ

#### 阻塞的实现

以ArrayList为例子，非阻塞的offer通过ReentrantLock实现，阻塞通过condition实现，condition底层通过AQS实现。
```java
final ReentrantLock lock;
private final Condition notFull;
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock(); 通过lock实现
        try {
            if (count == items.length)
                return false;
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }

    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```
