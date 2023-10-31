 
##### 标签：java,HashSet,TreeSet,去重,排序  
   
java中set的实现方式主要有HashSet和TreeSet两种方式，可以理解一个是二叉树实现，一个是哈希实现。   
  
#### 去重：  
HashSet的存和取都是O(1)，可通过HashSet来去重，去重的实体类的等价，可以通过重写类的hashCode和equals方法来实现。
代码实现如下：       

```    
NetTrack是一个互联网轨迹类，根据发生时间和合作方code去重校验。  
```  
 
```  
    @Override
    public int hashCode() {// 重写hashCode方法
        String hashFlag = this.partnerCode + this.exactTime;
        return hashFlag.hashCode();
    }

    @Override
    public boolean equals(Object obj) {// 重写equals方法
        if (this == obj) {
            return true;
        }
        if (null != obj && obj instanceof NetTrack) {
            NetTrack netTrack = (NetTrack) obj;
            // 用合作方，事件发生时间
            if (partnerCode.equals(netTrack.partnerCode) && exactTime.equals(netTrack.exactTime)) {
                return true;
            }
        }
        return false;
    }    
        
```   

#### 排序  
HashTree实现是二叉排序树，存取都是O(log(n)),可以高效的实现排序。实体类的排序可以通过，实现Comparable来实现排序，通过重写compareTo来实现。  
自然排序，按小到大的顺序，比较此对象与指定对象的顺序。如果该对象小于、等于或大于指定对象，则分别返回负整数、零或正整数。   
如果降序排列的话则相反。  
代码如下（按照发生日期排序）
  
```  
    @Override
    public int compareTo(NetTrack netTrack) {
        if (netTrack.getOccurTime() == null) {
            return -1;
        }
        if (netTrack.getOccurTime().compareTo(this.occurTime) > 0) {
            return 1;
        }
        return -1;
    }  
```     
