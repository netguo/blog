### 背景
最近开发中遇到一个问题，一些工具类采用了单例的方式，单例是通过内部静态类的方式实现。但是单例依赖spring中的单例，单例初始化时spring单例并没有初始化，会报ClassNotFound。进而学习了几种常见的单例的实现。
对于这个问题，依赖于spring的单例是在

### 主题

##### 饿汉方式  
利用static在类加载时初始化。线程安全的，会造成不必要的对象。

```  
public class Singleton {  
    private static Singleton instance = new Singleton();  
    private Singleton (){}  
    public static Singleton getInstance() {  
    return instance;  
    }  
}    

public class Singleton {  
    private Singleton instance = null;  
    static {  
    instance = new Singleton();  
    }  
    private Singleton (){}  
    public static Singleton getInstance() {  
    return this.instance;  
    }  
}    
```
##### 利用内部静态的方式实现
相对于饿汉模式，内部静态类在第一次使用时才初始化，懒加载。

```    
public class Singleton {  
    private static class SingletonHolder {  
        private static final Singleton INSTANCE = new Singleton();  
    }  
    private Singleton (){}  
    public static final Singleton getInstance() {  
        return SingletonHolder.INSTANCE;  
    }  
} 
```

##### 懒加载模式
```  
public class Singleton {  
    private static Singleton instance;  
    private Singleton (){}  
    public static synchronized Singleton getInstance() {  
    if (instance == null) {  
        instance = new Singleton();  
    }  
    return instance;  
    }  
} 
```  

##### 枚举方式
```  
public enum Singleton {  
    INSTANCE;  
    public void whateverMethod() {  
    }  
}    
```

### 其它问题

##### 序列化问题
当单例实现了java.io.Serializable时，单例会被破坏，Enum可以避免这个问题。有待研究。。。

