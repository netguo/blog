### 背景
ThreadLocal是在当前线程中保持一个线程中的值。之前没有仔细看过源码，今天code review时，代码中是在bean的一个单例中设置的，tomcat层，如果并发是通过线程池维护着线程，隶属于线程池中核心线程的，那么该线程的ThreadLocal会不会被反复利用。  
随后研究了一下ThreadLocal的源码，网上看了一些资料，原来还有弱引用，内存泄露的探讨。后来还有关于ThreadLocal是否用于解决线程安全问题讨论。
[这个博客，写的比较简单，评论中引起了较多的讨论](http://my.oschina.net/lichhao/blog/111362?p=2&temp=1470067119126)
[一个分析帖子，写的不错](http://qifuguang.me/2015/09/02/%5BJava%E5%B9%B6%E5%8F%91%E5%8C%85%E5%AD%A6%E4%B9%A0%E4%B8%83%5D%E8%A7%A3%E5%AF%86ThreadLocal/)     

总体结论是：   

* ThreadLocal是TLS(Thread Local Storage)，设计之初是为了存储线程的上下文信息，线程的局部变量。  
* 普通的ThreadLocal不会造成内存泄露，但是如果与线程池联合使用时，就可能会有这个问题。

### 分析   
先看ThreadLocal的存储，这个是讨论内存泄露的关键。    
每一个线程Thread中有一个属性，如下：  
ThreadLocal.ThreadLocalMap threadLocals = null;  
当前线程持有一个ThreadLocal.ThreadLocalMap，此时线程栈中还有一个ThreadLocal的对象地址。  
再看ThreadLocalMap的存储格式：    

```  
static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal> {

            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }  
}
```  

ThreadLocalMap的存储，以ThreadLocal为Key,取得value。然而对于Entry来讲，ThreadLocal是弱引用，在垃圾回收时ThreadLocal会被回收，此时Entry对应的为null,value。如果此时，如果线程不释放，那么key为null的object就不会被回收，而且key为null的将会越来越多，如果value属于大对象，那么更容易造成内存泄露。因为当这个线程get是，不存在变会重新为该线程创建threadlocalmap的entry实例，造成内存溢出。    

在getEntry，setEntry时会对key为null的进行回收。
### 源码分析：
ThreadLocal，我们通常用的方法是，get()，set()，先看在ThreadLocal源码中对于这两个方法的实现。     
   
    ---  

* _get和set的调用过程_  

```  
 1. get过程
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }    

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }  

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        //value是null，map是null都会调用，重新获取map
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }  

先获取当前线程，然后根据当前线程，获取ThreadLocalMap，可以看到getMap()方法是从当前线程获取该线程的ThreadLocal值。   
如果当前线程获取的ThreadLocalMap为空，或者对应的值为空，返回一个初始化的value，如果map为空，初始化一个map，并赋值到该线程。  

2. set过程
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }   
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }  

``` 
可以看到每一个线程都维护这自己的一个自己的ThreadLocalMap，维护着当前线程的的变量。  

---  
 
### 理解ThreadLocal的设计  
首先，还是开头那句话，ThreadLocal的设计是为了提供线程内部的局部变量，可以将一个公有变量编程私有。  

ThreadLocal本身是一个对象，本身是一个多线程共享的对象，但有自己的get，set方法，自身的get，set方法是获取、设置线程私有的对象，也就是说，ThreadLocal本身只是一个数据结构，真正的数据是维护在线程自己内部。  

ThreadLocal内部有一个静态内部类，维护着线程私有变量的数据结构。  

TLS解决了这样一个需求，就是希望在一个,每个线程的线程上下文,环境下执行的任何实体（函数，组件等）内，都能访问某一个变量。 那么很明显，这个变量是需要做成全局的，但是，普通全局的会污染别的线程。所以，此时需要TLS。
所以ThreadLocal是为了解决同一个线程中上下文共享问题。

### 思考
为什么这样设计，ThreadLocal本身是一个线程共享的对象，如果数据维护在自己的内部，那必然会有多线程问题，如果只是提供一个数据结构，具体的数据由线程自己维护。那么可以防止同步，避免并发问题。