### 问题  
  
##### 问题背景   
 
在findbug审核时，有段代码如下，提示synchronized应该去掉（flowStatisticCache是一个ConcurrentHashMap）。    

```  
    public void increaseFlow(String partnerCode, String serviceType) {
        String key = partnerCode + serviceType;
        if(flowStatisticCache.containsKey(key)){
            FlowStatisticCache info = flowStatisticCache.get(key);
            info.incrementUsedCount();
        }else{
            synchronized (flowStatisticCache) {
                if(flowStatisticCache.containsKey(key)){
                    FlowStatisticCache info = flowStatisticCache.get(key);
                    info.incrementUsedCount();
                }else{
                    FlowStatisticCache info = new FlowStatisticCache();
                    info.setPartnerCode(partnerCode);
                    info.setServiceType(serviceType);
                    info.incrementUsedCount();
                    flowStatisticCache.put(key, info);
                }
            }
        }
    }  
```    

然后就展开了讨论，从网上找资料讲ConcurrentHashMap是弱一致性的，不能保证取数据强一致性。所以在这里当flowStatisticCache.containsKey(key)为false时，再加锁判断是完全合理的。
 
 
资料如下：  
[为什么ConcurrentHashMap是弱一致的](http://ifeve.com/concurrenthashmap-weakly-consistent/)  
[深入分析ConcurrentHashMap](http://ifeve.com/concurrenthashmap/)  

__但是上述文章中用到的源码和我在jdk7中看到的不同，对源码进行阅读__  

containsKey实现方式跟get方法类似，可以参考下面代码get方法为什么不能保证原子性。isEmpty和size的代码类似，是弱一致性（这块代码比较有意思），写数据是怎么保证原子性可以参考put方法。


### ConcurrentHashMap  
ConcurrentHashMap实际上是利用一个分段锁的概念，提高并发性与伸缩性。 
特性：分段锁，写原子性，读弱一致性 
   
--- 

### 源码解析  

#### 构造函数：   
 
```  
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;  
    } 
    //ssize为concurrencyLevel,sshift为concurrencyLevel的指数位
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;  
    //c为initialCapacity/concurrencyLevel，向上取整
    if (c * ssize < initialCapacity) 
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;  
    //cap大小为大于c的最小2的指数的整数    
    // create segments and segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}    

```     

* 三个参数：  
    * initialCapacity：初始大小（默认是16）  
    * loadFactor：负载因子（默认0.75）
    * concurrencyLevel：分段大小（默认是16）
* 可以看出，初始化segmentShift为32-(concurrencyLevel的2的指数位)，segmentMask为concurrencyLevel-1  
* 初始化Segment数组，大小为concurrencyLevel，并顺序的添加到内存中 
  
Segment是ConcurrentHashMap真正存取数据的地方，也就是ConcurrentHashMap的所谓的锁分段实现方式，构造函数如下：    

``` 
static final class Segment<K,V> extends ReentrantLock implements Serializable {  
      Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
            this.loadFactor = lf;
            this.threshold = threshold;
            this.table = tab;
        }
}
```    
可以看到，Segment实现了ReentrantLock，真正的锁也是在Segment通过ReentrantLock实现的锁。

#### get操作    

```
public V get(Object key) {
  Segment<K,V> s; // manually integrate access methods to reduce overhead
  HashEntry<K,V>[] tab;
  int h = hash(key);
  long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
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

* 可以看出，最终取数据时，是通过UNSAFE.getObjectVolatile取数据的。volatile是只能保证可见性，不能保证原子性的，所以get操作是弱一致性的。  

#### put操作    

```  
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
```    

前面的hash操作，可以先不看，最终是先hash到Segment，然后执行Segment中的put操作，真正的加锁是在Segment中加锁，也就真正实现了锁分段。如下：  

```  
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
   //获取一个锁，获取失败通过scanAndLockForPut获取
   HashEntry<K,V> node = tryLock() ? null :
      scanAndLockForPut(key, hash, value);
   V oldValue;
   try {
       HashEntry<K,V>[] tab = table;
       //重新计算hash
       int index = (tab.length - 1) & hash;
       //根据hash计算出第一个节点
       HashEntry<K,V> first = entryAt(tab, index);
       //从first开始遍历邻接链表
       for (HashEntry<K,V> e = first;;) {
           if (e != null) {
             K k;
             f ((k = e.key) == key ||
               (e.hash == hash && key.equals(k))) {
                   oldValue = e.value;
                   //onlyIfAbsent，false则替换原来节点
                   if (!onlyIfAbsent) {
                       e.value = value;
                       ++modCount;  //modCount代表segment改变次数
                   }
                   break;
             }
             e = e.next;
          }
          else { //当hash在table中还没有存在数据
             if (node != null)
             	//利用scanAndLockForPut随机创建的node
               node.setNext(first);
             else
               node = new HashEntry<K,V>(hash, key, value, first);
             int c = count + 1;  
               //达到阀值重新hash
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
       //finally中最后释放锁，ReentrantLock，必须有此操作啊！！
      unlock();
    }
    return oldValue;
  }  
```     
 
* put操作，首先会获取一个锁，如果tryLock()，失败，则调用scanAndLockForPut()获取锁  
* 获取锁后的更新操作比较简单，比较有意思的是modCount，modCount会在size和isEmpty中用到。   
* 根据锁最终达到了线程安全  
* 当segment中该hash处没有值是，scanAndLockForPut会投机的创建一个node，下个节点设置为null即可。

```  
        private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) { 
        		//根据hash值计算出，该hash值对应的第一个元素。（HashMap碰撞用邻接链表实现）
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            HashEntry<K,V> node = null;
            int retries = -1; // negative while locating node  
            //如果获取到锁，直接返回node为null
            while (!tryLock()) {
                HashEntry<K,V> f; // to recheck first below  
                //在邻接链表中找到对应的结点，找到retries置为0
                if (retries < 0) {
                    if (e == null) {
                        if (node == null) // speculatively create node
                            node = new HashEntry<K,V>(hash, key, value, null);
                        retries = 0;
                    }
                    else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                //MAX_SCAN_RETRIES（如果JVM处理器数量大于1为64，等于1为1）
                //retries次数大于MAX_SCAN_RETRIES，则lock()
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                //retries为0且没有改变，如果有改变，retries重新赋值为-1，e,first重新指向新的头结点
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f; // re-traverse if entry changed
                    retries = -1;
                }
            }
            return node;
        }  
```     
 
* 可以看出，scanAndLockForPut会一直tryLock，直到获取到锁，或者retries达到MAX\_SCAN\_RETRIES，调用lock方法。  
* lock方法，如果锁被占有，则线程挂起，等待锁被释放。  
* 当头结点为空时，该方法还会投机的根据该key创建一个节点，在put中用到。

#### size方法  
刚刚有提到过modCount会在size中用到，这个方法比较有意思。    

```  
public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits
        long sum;         // sum of modCounts
        long last = 0L;   // previous sum
        int retries = -1; // first iteration isn't retry
        try {
            for (;;) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L;
                size = 0;
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }  
```    

* 为保证高效，retries次数为RETRIES\_BEFORE\_LOCK之前，不进行加锁操作，利用方法刚刚开始的所有的Segment的modcount总数与，方法最后的modcount数目是否相同来比较，如果相同，则返回总数，不同则retry。
* 如果retry次数超过RETRIES\_BEFORE\_LOCK，则加锁  
* __其实sum == last并不能保证数据的强一致性的，所以size也是若一致性的。__




   

 



