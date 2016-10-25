
摘抄自redis的官网[redisdoc](http://redisdoc.com/index.html)，官网doc不方便查看，如果想看一个语句还需要点击进入子页面，摘抄一下方便工作。  

### redis操作总结    

#### key: 
  
+ DEL: key [key...]。 删除给定一个或多个key  
+ DUMP: key。  序列化给定的key  
+ EXISTS:  检查给定的KEY是否存在  
+ EXPIRE:  key secouds。 为给定的key设置生存时间，通过DEL移除或者SET,GETSET复写    
+ PEXPIRE： key milliseconds。设置过期时间，单位为毫秒
+ EXPIREAT:  key timestamp，与	EXPIRE类似，参数为UNIX时间戳    
+ PEXPIREAT：key milliseconds-timestamp。毫秒级的UNIX时间戳 
KEYS：pattern。查找所有符合给定模式 pattern 的 key。特殊符号用\隔开。  
+ MIGRATE：host port key destination-db timeout [COPY] [REPLACE]。将 key 原子性地从当前实例传送到目标实例的指定数据库上，一旦传送成功， key 保证会出现在目标实例上，而当前实例上的 key 会被删除。  
+ MOVE：key db。将当前数据库的 key 移动到给定的数据库 db 当中。  
+ OBJECT： subcommand [arguments [arguments]]。从内部察看给定key的Redis对象。  
OBJECT 命令有多个子命令：

   * OBJECT REFCOUNT <key> 返回给定 key 引用所储存的值的次数。此命令主要用于除错。
   * OBJECT ENCODING <key> 返回给定 key 锁储存的值所使用的内部表示(representation)。
   * OBJECT IDLETIME <key> 返回给定 key 自储存以来的空闲时间(idle， 没有被读取也没有被写入)，以秒为单位。  

+ PERSIST: key。 移除一个key的过期时间，讲key转为持久型。   
+ PTTL：key。 它以毫秒为单位返回 key 的剩余生存时间，以秒为单位。   
+ TTL：key。 以秒为单位，返回给定 key 的剩余生存时间。  
+ RANDOMKEY： 从当前数据库中随机返回(不删除)一个 key  
+ RENAME：key newkey。将 key 改名为 newkey 。
+ RENAMENX：key newkey。当且仅当 newkey 不存在时，将 key 改名为 newkey。  
+ RESTORE：key ttl serialized-value [REPLACE]。反序列化给定的序列化值，并将它和给定的key关联。  
+ SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC | DESC] [ALPHA] [STORE destination]。返回或保存给定列表、集合、有序集合 key 中经过排序的元素。  
+ TYPE : key。返回 key 所储存的值的类型。  
+ SCAN ：cursor [MATCH pattern] [COUNT count] 。SCAN 命令及其相关的 SSCAN 命令、 HSCAN 命令和 ZSCAN 命令都用于增量地迭代（incrementally iterate）一集元素（a collection of elements）：

  * SCAN 命令用于迭代当前数据库中的数据库键。
  * SSCAN 命令用于迭代集合键中的元素。
  * HSCAN 命令用于迭代哈希键中的键值对。
  * ZSCAN 命令用于迭代有序集合中的元素（包括元素成员和元素分值）。  
  
 
### String    
 
+ APPEND： key value，讲value追加到key原来的值的末尾。  
+ BITCOUNT： key [start] [end]。计算给定字符串中，被设置为 1 的比特位的数量。可以用于统计频率。  
+ BITOP： operation destkey key [key ...]。对一个或多个保存二进制位的字符串 key 进行位元操作，并将结果保存到 destkey 上。AND OR XOR NOT  
+ DECR ：key。将key中储存的数字值减一。  
+ DECRBY： key decrement。将 key 所储存的值减去减量 decrement 。  
+ GET：key  返回key所关联的字符串值。（只能处理字符串值）。 
+ GETBIT：key offset。对key所储存的字符串值，获取指定偏移量上的位(bit)。   
+ GETRANGE： key start end。返回 key 中字符串值的子字符串，字符串的截取范围由 start 和 end 两个偏移量决定(包括 start 和 end 在内)。  
+ INCR：key。讲key中存储的数字值增一。  
+ INCRBY：key increment。将key所储存的值加上增量 increment。  
+ INCRBYFLOAT： key increment。为 key 中所储存的值加上浮点数增量 increment 。  
+ MGET： key [key ...]。返回所有(一个或多个)给定 key 的值。  
+ MSET： key value [key value ...]。同时设置一个或多个 key-value对。  
+ MSETNX key value [key value ...]。同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。  
+ PSETEX key milliseconds value。这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间。  
+ SET：key value [EX seconds] [PX milliseconds] [NX|XX]。将字符串值 value 关联到 key 。  
+ SETBIT：key offset value。对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)。  
+ SETEX key seconds value。将值 value 关联到 key ，并将 key 的生存时间设为 seconds (以秒为单位)。  
+ SETNX key value。将key的值设为value ，当且仅当key不存在。  
+ SETRANGE：key offset value。用 value 参数覆写(overwrite)给定 key 所储存的字符串值，从偏移量 offset 开始。    
+ STRLEN： key。返回key所储存的字符串值的长度。    

### Hash    
 
+ HDEL key field [field ...]。删除哈希表 key 中的一个或多个指定域，不存在的域将被忽略。  
+ HEXISTS key field。查看哈希表 key 中，给定域 field 是否存在。  
+ HGET：key field。返回哈希表 key 中给定域 field 的值。  
+ HGETALL： key。返回哈希表 key 中，所有的域和值。  
+ HINCRBY： key field increment。为哈希表 key 中的域 field 的值加上增量 increment 。  
+ HINCRBYFLOAT： key field increment。为哈希表 key 中的域 field 加上浮点数增量 increment。  
+ HKEYS： key。返回哈希表 key 中的所有域。  
+ HLEN： key。返回哈希表 key 中域的数量。  
+ HMGET： key field [field ...]。返回哈希表 key 中，一个或多个给定域的值。  
+ HSET：key field value。将哈希表 key 中的域 field 的值设为 value。  
+ HSETNX： key field value。将哈希表 key 中的域 field 的值设置为 value ，当且仅当域 field 不存在。  
+ HVALS： key。返回哈希表 key 中所有域的值。  
+ HSCAN： key cursor [MATCH pattern] [COUNT count]。迭代哈希键中的键值对。  

### List    
 
+ LPOP：key 移除并返回列表 key 的头元素。
+ BLPOP： key [key ...] timeout。 是列表的阻塞式(blocking)弹出原语。弹出头部。  
 模式：事件提醒。  
 
+ BRPOP： key [key ...] timeout。BRPOP 是列表的阻塞式(blocking)弹出原语。弹出尾部。  
+ BRPOPLPUSH： source destination timeout。  
+ LINDEX： key index。返回列表 key 中，下标为 index 的元素。  
+ LINSERT： key BEFORE|AFTER pivot value。将值 value 插入到列表 key 当中，位于值 pivot 之前或之后。  
+ LLEN： key。返回列表 key 的长度。  
+ LPUSH： key value [value ...]。将一个或多个值 value 插入到列表 key 的表头  
+ RPUSH：RPUSH key value [value ...]。将一个或多个值 value 插入到列表 key 的表尾(最右边)
+ LPUSHX： key value。将值 value 插入到列表 key 的表头，当且仅当 key 存在并且是一个列表。    
+ RPUSHX： key value。将值 value 插入到列表 key 的表尾，当且仅当 key 存在并且是一个列表。
+ LRANGE： key start stop。返回列表 key 中指定区间内的元素，区间以偏移量 start 和 stop 指定。  
+ LREM： key count value。根据参数 count 的值，移除列表中与参数 value 相等的元素。  
+ LSET： key index value。将列表key下标为 index 的元素的值设置为value。  
+ LTRIM： key start stop。对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。  
+ RPOPLPUSH ：source destination。命令 RPOPLPUSH 在一个原子时间内，执行以下两个动作：  
 
      *  将列表 source 中的最后一个元素(尾元素)弹出，并返回给客户端。
      *  将 source 弹出的元素插入到列表 destination ，作为 destination 列表的的头元素。    
  
     模式： 安全的队列，循环列表   
+ BRPOPLPUSH： source destination timeout。BRPOPLPUSH 是 RPOPLPUSH 的阻塞版本，当给定列表 source 不为空时， BRPOPLPUSH 的表现和 RPOPLPUSH 一样。当列表 source 为空时， BRPOPLPUSH 命令将阻塞连接，直到等待超时，或有另一个客户端对 source 执行 LPUSH 或 RPUSH 命令为止。  

### Set  
+  SADD： key member [member ...]。将一个或多个 member 元素加入到集合 key 当中，已经存在于集合的 member 元素将被忽略。  
+  SCARD： key。返回集合 key 的基数(集合中元素的数量)。  
+  SDIFF： key [key ...]。返回一个集合的全部成员，该集合是所有给定集合之间的差集。不存在的 key 被视为空集。  
+  SDIFFSTORE： destination key [key ...]。这个命令的作用和 SDIFF 类似，但它将结果保存到 destination 集合，而不是简单地返回结果集。  
+  SINTER： key [key ...]。返回一个集合的全部成员，该集合是所有给定集合的交集。不存在的 key 被视为空集。  
+  SINTERSTORE： destination key [key ...]。这个命令类似于 SINTER 命令，但它将结果保存到 destination 集合，而不是简单地返回结果集。  
+  SISMEMBER： key member。判断 member 元素是否集合 key 的成员。  
+  SMEMBERS： key。返回集合 key 中的所有成员。不存在的 key 被视为空集合。  
+  SMOVE： source destination member。将member元素从source集合移动到destination集合。原子操作。  
+  SPOP： key。移除并返回集合中的一个随机元素。  
+  SRANDMEMBER： key [count]。如果命令执行时，只提供了 key 参数，那么返回集合中的一个随机元素。从 Redis 2.6 版本开始， SRANDMEMBER 命令接受可选的 count 参数：返回个数。  
+  SREM： key member [member ...]。移除集合 key 中的一个或多个 member 元素，不存在的 member 元素会被忽略。  
+  SUNION： key [key ...]。返回一个集合的全部成员，该集合是所有给定集合的并集。不存在的 key 被视为空集。  
+  SUNIONSTORE： destination key [key ...]。这个命令类似于 SUNION 命令，但它将结果保存到 destination 集合，而不是简单地返回结果集。  
+  SSCAN： key cursor [MATCH pattern] [COUNT count]。参考SCAN。  

### SortedSet  
+ ZADD: key score member [[score member] [score member] ...]。将一个或多个 member 元素及其 score 值加入到有序集 key 当中。  
+ ZCARD： key。返回有序集 key 的基数。  
+ ZCOUNT： key min max。返回有序集 key 中， score 值在 min 和 max 之间(默认包括 score 值等于 min 或 max )的成员的数量。  
+ ZINCRBY： key increment member。为有序集 key 的成员 member 的 score 值加上增量 increment。  
+ ZRANGE： key start stop [WITHSCORES]。返回有序集 key 中，指定区间内的成员。其中成员的位置按 score 值递增(从小到大)来排序。  
+ ZRANK： key member。返回有序集 key 中成员 member 的排名。其中有序集成员按 score 值递增(从小到大)顺序排列。  
+ ZREM： key member [member ...]。移除有序集 key 中的一个或多个成员，不存在的成员将被忽略。   
+ ZREMRANGEBYRANK： key start stop。移除有序集 key 中，指定排名(rank)区间内的所有成员。 
+ ZREMRANGEBYSCORE： key min max。移除有序集 key 中，所有 score 值介于 min 和 max 之间(包括等于 min 或 max )的成员。  
+ ZREVRANGE： key start stop [WITHSCORES]。返回有序集 key 中，指定区间内的成员。  
+ ZREVRANGEBYSCORE： key max min [WITHSCORES] [LIMIT offset count]。返回有序集 key 中， score 值介于 max 和 min 之间(默认包括等于 max 或 min )的所有的成员。有序集成员按 score 值递减(从大到小)的次序排列。  
+ ZREVRANK： key member。返回有序集 key 中成员 member 的排名。其中有序集成员按 score 值递减(从大到小)排序。  
+ ZSCORE： key member。返回有序集 key 中，成员 member 的 score 值。  
+ ZUNIONSTORE： destination numkeys key [key ...] [WEIGHTS weight [weight ...]][AGGREGATE SUM|MIN|MAX]。计算给定的一个或多个有序集的并集，其中给定 key 的数量必须以 numkeys 参数指定，并将该并集(结果集)储存到 destination 。默认情况下，结果集中某个成员的 score 值是所有给定集下该成员 score 值之 和。  
   * WEIGHTS：权重因子  
   * AGGREGATE：聚合运算      
+ ZINTERSTORE： destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]。计算给定的一个或多个有序集的交集，其中给定 key 的数量必须以 numkeys 参数指定，并将该交集(结果集)储存到 destination 。  
+ ZSCAN： key cursor [MATCH pattern] [COUNT count]。迭代输出，参看SCAN。  
+ ZRANGEBYLEX： key min max [LIMIT offset count]。当有序集合的所有成员都具有相同的分值时， 有序集合的元素会根据成员的字典序（lexicographical ordering）来进行排序， 而这个命令则可以返回给定的有序集合键 key 中， 值介于 min 和 max 之间的成员。  
+ ZLEXCOUNT： key min max。对于一个所有成员的分值都相同的有序集合键 key 来说， 这个命令会返回该集合中， 成员介于 min 和 max 范围内的元素数量。  
+ ZREMRANGEBYLEX： key min max。对于一个所有成员的分值都相同的有序集合键 key 来说， 这个命令会移除该集合中， 成员介于 min 和 max 范围内的所有元素。  

###  HyperLogLog（基数计算，用来统计个数）
  
+ PFADD： key element [element ...]。将任意数量的元素添加到指定的 HyperLogLog 里面。  
+ PFCOUNT： key [key ...]。当 PFCOUNT 命令作用于单个键时， 返回储存在给定键的 HyperLogLog 的近似基数， 如果键不存在， 那么返回 0 。  
+ PFMERGE： destkey sourcekey [sourcekey ...]。将多个 HyperLogLog 合并（merge）为一个 HyperLogLog ， 合并后的 HyperLogLog 的基数接近于所有输入 HyperLogLog 的可见集合（observed set）的并集。
