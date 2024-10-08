### 一、Redis是什么？
Redis（Remote Dictionary Server）是一个使用ANSI C编写的开源、支持网络、基于内存、分布式、可选持久性的键值对存储数据库。其性能高，支持丰富的数据结构，并提供多种语言API，根据月度排行网站DB-Engines.com的数据，Redis是最流行的键值对存储数据库。
#### Redis可以用来做什么？
1、缓存
几乎所有的大型网站都有缓存机制，提供更快的访问速度，并减轻数据库的访问压力，MySQL写并发600，读并发2000，Redis官方单台能达到10W并发。Redis是市面上最流行的缓存数据库，相比与Memcached，提供了更丰富的数据结构，并提供了持久化功能。
2、排行榜和计数器
Redis 的数据结构如有序集合和整数类型，使其非常适合用于存储排行榜数据和进行计数操作。
3、分布式锁
在分布式系统中，用于协调对共享资源的访问，可以用Redis的原子性操作命令来实现。
4、会话存储
各个机器共享用户Session
5、队列
Redis的list、set操作，可以用于实现队列功能；提供发布订阅功能实现消息队列；Stream功能实现轻量级的消息队列。
### 二、Redis的数据结构
Redis的基本数据模型是key-value数据集合，value类型包括了String、哈希表、列表、集合、有序集合等。
#### 1、全局命令

- keys：查看所有键，O(n)
- dbsize：当前数据库键的总值，Redis内置键的总数，O(1)
- exists：检查键是否存在
- del：删除键
- expire：键过期，秒级
- pexpire key millseconds：键过期，毫秒级
- expireat key timestamp：在时间戳(秒级)后过期
- pexpireat key timestamp：在时间戳(毫秒级)后过期
- type：键的数据类型
- randomkey：随机返回一个键
- rename：键重命名
#### 2、基本数据类型
##### 2.1 String字符串
Redis 字符串存储字节序列，包括文本、序列化对象和二进制数组。因此，字符串是可以与 Redis 键关联的最简单的值类型。它们通常用于缓存，但它们支持附加功能，使您也可以实现计数器并执行按位运算。
常用命令：

- set key value [ex seconds｜px millisecends] [nx | xx]  nx键必须不存在才可以设置成功，xx键必须存在用于更新。
- get 获取值，不存在返回 nil
- mset 批量设置值
- mget 批量获取值
- incr自增，decr自减，incrby自增指定数字，decrby自减指定数字，incrbyfloat自增浮点数。
- append追加
- strlen字符串长度
- ......
##### 2.2 Hash哈希
Redis 哈希是结构化为字段值对集合的记录类型。可以使用哈希来表示基本对象并存储计数器分组等。虽然哈希可以方便地表示_对象_，但实际上，可以放入哈希中的字段数量没有实际限制（可用内存除外）。
常用命令：

- hset key field value
- hget key field
- hdel key field
- hmset 批量
- hmget 批量
- ......
##### 2.3、List列表
Redis 列表是字符串值的链接列表。 Redis 列表经常用于：

- 实现堆栈和队列。
- 为后台工作系统构建队列管理。

基本命令：

- LPUSH将新元素添加到列表的头部；RPUSH添加到尾巴。
- LPOP从列表头部删除并返回一个元素；RPOP做同样的事情，但是从列表的尾部开始。
- LLEN返回列表的长度。
- LMOVE以原子方式将元素从一个列表移动到另一个列表。
- LTRIM将列表缩小到指定的元素范围。

阻塞命令：

- BLPOP从列表头部删除并返回一个元素。如果列表为空，则该命令将阻塞，直到有元素可用或达到指定的超时为止。
- BLMOVE以原子方式将元素从源列表移动到目标列表。如果源列表为空，该命令将阻塞，直到有新元素可用。
##### 2.4 Set集合

- SADD将新成员添加到集合中。
- SREM从集合中删除指定的成员。
- SISMEMBER测试字符串的集合成员资格。
- SINTER返回两个或多个集合共有的成员集（即交集）。
- SCARD返回集合的大小（也称为基数）。

2.5、ZSET有序集合
排序集是按关联分数排序的唯一字符串（成员）的集合。当多个字符串具有相同分数时，字符串按字典顺序排序。排序集的一些用例包括：

- 排行榜。例如，您可以使用排序集轻松维护大型在线游戏中最高分数的有序列表。
- 速率限制器。特别是，您可以使用排序集构建滑动窗口速率限制器，以防止过多的 API 请求。

可以将排序集视为集合和哈希的混合。虽然集合内的元素没有排序，但排序集合中的每个元素都与一个浮点值相关联，称为分数。
常用命令：

- zadd 添加成员
- zcard 计算成员个数
- zscore 计算某成员分数
- zrank 计算成员排名
- zrem 删除成员
- zincrby 增加成员分数
- zrange和zrevrange返回指定排名范围的成员
- zrangebyscore返回指定分数范围的成员
- zcount 返回指定分数范围成员个数
- zremrangebyrank 按升序删除指定排名内的元素
- ......
#### 3、高级数据结构
##### 3.1 位图Bitmap
位图不是实际的数据类型，而是在 String 类型上定义的一组面向位的操作，将其视为位向量。由于字符串是二进制安全 blob，其最大长度为 512 MB，因此它们适合设置最多 2^32 个不同位。
常用命令：

- SETBIT命令将位编号作为其第一个参数，将要设置该位的值作为其第二个参数，即1或0。
- GETBIT仅返回指定索引处的位的值。
- BITOP在不同字符串之间执行按位运算。提供的运算有 AND、OR、XOR 和 NOT。
- BITCOUNT执行总体计数，报告设置为1的位数。
- BITPOS查找具有指定值0或1的第一位。
##### 3.2 布隆过滤器
1970 年布隆提出了一种布隆过滤器的算法，用来判断一个元素是否在一个集合中。
本质上布隆过滤器是一种数据结构，比较巧妙的概率型数据结构（probabilistic data structure），特点是高效地插入和查询，可以用来告诉你 “某样东西一定不存在或者可能存在”。
相比于传统的 List、Set、Map 等数据结构，它更高效、占用空间更少，但是缺点是其返回的结果是概率性的，而不是确切的。
布隆过滤器广泛应用于网页黑名单系统、垃圾邮件过滤系统、爬虫网址判重系统等，Google 著名的分布式数据库 Bigtable 使用了布隆过滤器来查找不存在的行或列，以减少磁盘查找的IO次数，Google Chrome浏览器使用了布隆过滤器加速安全浏览服务。
如果hash冲突增多，那么会增大误判率，解决方案：采用K个哈希函数，分别标识；增大数组长度。数组长度、哈希函数个数、元素个数、误判率有一个具体公示。
##### 3.3 HyperLogLog
HyperLogLog基于概率论中伯努利试验并结合了极大似然估算方法，并做了分桶优化。用于估计集合的基数。作为一种概率数据结构，HyperLogLog 以完美的准确性换取高效的空间利用，常用于去重后个数，例如UV、在线用户数等。
Redis HyperLogLog 实现最多使用 12 KB，并提供 0.81% 的标准错误。
基本命令

- PFADD将一个项目添加到 HyperLogLog。
- PFCOUNT返回集合中项目数量的估计值。
- PFMERGE将两个或多个 HyperLogLog 合并为一个。
##### 3.4 GEO
地理空间索引可存储坐标并搜索它们。对于查找给定半径或边界框内的附近点非常有用。
常用命令

- [GEOADD](https://redis.io/commands/geoadd)将位置添加到给定的地理空间索引。
- [GEOSEARCH](https://redis.io/commands/geosearch)返回具有给定半径或边界框的位置。
- GEODIST获取两个Key的距离
### 三、Redis的高级功能
#### 1、发布订阅
Redis主要提供了发布消息、订阅频道、取消订阅以及按照模式订阅和取消订阅等命令。
发布消息   publish channel message 
返回值是接收到信息的订阅者数量，如果是0说明没有订阅者，这条消息就丢了。
订阅消息   subscribe channel [channel ...]
订阅者可以订阅一个或多个频道，如果此时另一个客户端发布一条消息，当前订阅者客户端会收到消息。
使用场景：
需要消息解耦又并不关注消息可靠性的地方都可以使用发布订阅模式。如果消费者宕机重启，期间发送的消息都会丢失。
#### 2、流Stream
Redis的流是一种数据结构，其作用类似于append-only日志，但也实现了多种操作来克服典型append-only日志的一些限制。该特性如果是监听日志的变化，可以理解是一个新的强大的消息队列，并支持多播的可持久化，Redis的作者声明Stream地借鉴了Kafka的设计。
数据结构：ID+Message方式顺序存储。ID建议采用Redis自生成的模式，毫秒时间戳-整数自增。
写数据：xadd 
读数据：读数据可以三种理解 ，可以作为一个日志文件，tail -f形式不断读取文件末尾更新内容；时间序列存储，可以支持按照时间范围查询；消息队列形式。
常用命令：

- XADD 添加一个新的消息体到stream.
- XREAD 给定一个具体的位置，读取位置后续消息.
- XRANGE 两个ID之间的消息.
- XLEN stream的长度.
- XGROUP 添加消费者分组 

消息队列 ：
可以根据XREAD 指定消费的stream、起始ID，持续消息消息。xreadgroup可以指定分组，进行组内消费。同时还支持消费者ACK机制，是不是有点kafka的感觉了？
stream为每一个消费者分组维护了一个last_delivered_id，分组中任意一个消费者读取消息后，last_delivered_id都会后移。消费者 (Consumer) （非客户端）内部会有个状态变量pending_ids，它记录了当前已经被客户端读取，但是还没有 ack的消息。
stream作为消息队列有哪些问题呢？

- stream大小：可以通过maxlen设置一个最大大小，旧的消息会被删除。
- PEL如何保证消息不丢失：客户端如果断开链接，重新启动后，会消费PEL中消息和last_delivered_id后的消息。
- ACK是否可以不提交？：如果ACK不提交，那么pending_ids会占用大量内存，客户端消费后需要尽快提交 。
- 死信问题：如果一个消息一直不能消费成功，通过XDEL删除该消息，并ACK确认该消息 。
- 分区：redis的的stream并没有实现分区功能，可以创建多个steam自己实现分区。
- 高可用：依赖于高可用模型，哨兵或者集群。
- 数据一致性 ：依赖于redis的数据一致性，宕机情况下会存在数据丢失 。

通常并不建议采用redis的消息队列方案，采用更为成熟的kafka、RabbitMQ等专业的MQ，不过如果可以容忍redis消息队列的缺点，公司没有成熟的MQ基建的情况下，可以采用。
#### 3、lua脚本
Redis 2.6 版本通过内嵌支持 Lua 环境。也就是说一般的运用，是不需要单独安装Lua的。
通过使用LUA脚本：

- 减少网络开销，在Lua脚本中可以把多个命令放在同一个脚本中运行；
- 原子操作，redis会将整个脚本作为一个整体执行，中间不会被其他命令插入（Redis执行命令是单线程）。
- 复用性，客户端发送的脚本会永远存储在redis中，这意味着其他客户端可以复用这一脚本来完成同样的逻辑。

常用命令：
EVAL，执行一段脚本，EVAL script numkeys key [key ...] arg [arg ...]
示例：eval "return redis.call('mset',KEYS[1],ARGV[1],KEYS[2],ARGV[2])" 2 key1 key2 first second
EVALSHA，执行缓存在服务端的脚本
SCRIPT，上传/清除/Kill等脚本
#### 4、Pipeline
以管道形式，批量提交命令到服务端 ，用于减少网络传输消耗，虽然redis已经提供了mget,mset等命令，但是命令支持有限。
每次拼装的命令不是无限制的，一方面会增加客户端的等待时间，另一方面会造成一定的网络阻塞,可以将一次包含大量命令的Pipeline拆分成多次较小的Pipeline来完成，比如可以将Pipeline的总发送大小控制在内核输入输出缓冲区大小之内或者控制在单个TCP 报文最大值1460字节之内。
#### 5、事务
Redis通过MULTI、DISCARD、EXEC和WATCH四个命令来实现事务功能，将一组需要一起执行的命令放到multi和exec两个命令之间，multi 命令代表事务开始，exec命令代表事务结束，discard命令是回滚。
通过multi、exec、discard实现了简单的事物的原子性（一组动作要不执行，要不不执行），但是要注意Redis的事务功能很弱。在事务回滚机制上，Redis只能对基本的语法错误进行判断。Redis的的事物是不支持嵌套的。
有些应用场景需要在事务之前，确保事务中的key没有被其他客户端修改过，才执行事务，否则不执行。Redis 提供了watch命令来解决这类问题。
Redis是否真的实现了事务？？？
事务的属性ACID，原子性、隔离性、持久性，这个三个特性用于保证一致性。
隔离性：Redis是单线程执行的，可以保证。
原子性：虽然Redis的事物命令可以保证一批命令执行或者回滚，但是如果出现命令执行到一半机器宕机恢复，并不能保证后续命令继续执行/回滚。
持久性：Redis虽然有持久性功能，并不能保证数据不丢失
所以Redis是肯定不能保证数据一致性的，Redis内心独白（老子是个缓存，你让我保证一致性？？？）
#### 6、慢查询
Redis的慢查询只针对命令执行时间，不包括网络传输和队列中等待时间。
slowlog-log-slower-than就是时间预设阀值，slowlog-max-len慢查询日志存储大小。
slowlog-log-slower-than配置建议：默认值超过10毫秒判定为慢查询，需要根据Redis并发量调整该值。例如系统的应用场景，TPS是1000，那么建议1ms就可以作为慢查询。
### 四、Redis的设计与实现
#### 1、全局哈希表与渐进式rehash
Redis的存储是采用Hash存储，Hash冲突采用链表存储，数组+链表形式。可以理解为具体的KV存储，对于集合类的数据类型，value是指向集合的指针。
当hash冲突到达一定的阈值，需要对数组进行扩容，扩容后所有的数据需要rehash。那么问题来了，redis是单线程处理数据读写的，全局rehash消耗时间较长，会阻塞正常的数据读写。redis采用渐进式hash来保证。
Redis存在两张全局性哈希表，rehash过程，Redis 仍然正常处理客户端请求，每处理一个请求时，从哈希表 1 中的第一个索引位置开始，顺带着将这个索引位置上的所有 entries 拷贝到哈希表 2 中；等处理下一个请求时，再顺带拷贝哈希表 1 中的下一个索引位置的 entries。
#### 2、底层数据结构
![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1711537442455-5e08f7f8-0d5d-461d-a43f-87acf24831c5.png#averageHue=%23f3f3e0&clientId=u24816202-f2e2-4&from=paste&height=594&id=uda946970&originHeight=1188&originWidth=4000&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1045470&status=done&style=none&taskId=u19d5ec20-06d5-49bf-a0e6-441e2ca49c5&title=&width=2000)
**动态字符串：**
Redis没有采用C的原始字符串，自己设计一套，只支持部分string.h库中函数的数据结构，性能更高，更符合Redis的使用场景。
**压缩列表：**
压缩列表在表头有三个字段 zlbytes、zltail 和 zllen，分别表示列表长度、列表尾的偏移量和列表中的 entry 个数；压缩列表在表尾还有一个 zlend。
entry中有三个字段，prevlen：前一个entry的大小；encoding：不同的情况下值不同，用于表示当前entry的类型和长度；entry-data：用于存储entry表示的数据。
ziplist节省内存是相对于普通的list来说的，如果是普通的数组，那么它每个元素占用的内存是一样的且取决于最大的那个元素。所以ziplist在设计时就很容易想到要尽量让每个元素按照实际的内容大小存储，所以增加encoding字段，针对不同的encoding来细化存储大小；
**跳表：**
利用多级索引，实现O(log n)级别查找，用于排序集

**整数数组和压缩列表在查找时间复杂度方面并没有很大的优势，那为什么 Redis 还会把它们作为底层数据结构呢？**

- 内存利用率，数组和压缩列表都是非常紧凑的数据结构，它比链表占用的内存要更少。Redis是内存数据库，大量数据存到内存中，此时需要做尽可能的优化，提高内存的利用率。
- 数组对CPU高速缓存支持更友好，所以Redis在设计时，集合数据元素较少情况下，默认采用内存紧凑排列的方式存储，同时利用CPU高速缓存不会降低访问速度。当数据元素超过设定阈值后，避免查询时间复杂度太高，转为哈希和跳表数据结构存储，保证查询效率。
#### 3、线程和IO模型
![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1711959113791-4ffe0d0f-48c3-4114-8b79-8fd56119c0cc.png#averageHue=%23f4f6e7&clientId=ue5e5e5f3-54ae-4&from=paste&height=1125&id=u71a29246&originHeight=2250&originWidth=3472&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1256881&status=done&style=none&taskId=u0ecfd9e9-7724-44c4-b492-17e44b958a1&title=&width=1736)
Redis是单线程，主要是指Redis的网络IO和键值对读写是由一个线程来完成的，这也是Redis对外提供键值存储服务的主要流程。但Redis的其他功能，比如持久化、异步删除、集群数据同步等，其实是由额外的线程执行的。
Redis网络框架调用epoll机制，让内核监听这些套接字。select/epoll一旦监测到FD上有请求到达时，就会触发相应的事件。这些事件会被放进一个事件队列，Redis单线程对该事件队列不断进行处理。
**Redis6.0多线程**
Redis将所有数据放在内存中，内存的响应时长大约为100纳秒，对于小数据包，Redis服务器可以处理80,000到100,000 QPS，这也是Redis处理的极限了，对于80%的公司来说，单线程的Redis已经足够使用了。但随着越来越复杂的业务场景，有些公司动不动就上亿的交易量，因此需要更大的QPS。Redis 6 引入的多线程 IO 特性对性能提升至少是一倍以上。
#### 4、淘汰策略
Redis提供了8种淘汰策略：

- 不进行数据淘汰的策略，noeviction
- 根据过期时间进行淘汰：volatile-ttl(按时间)、volatile-random、volatile-lru、volatile-lfu
- 在所有数据范围内进行淘汰：allkeys-lru、allkeys-random、allkeys-lfu

**LRU，Least Recently Used（最近最少使用）**

在Redis中，LRU算法被做了简化，以减轻数据淘汰对缓存性能的影响。具体来说，Redis默认会记录每个数据的最近一次访问的时间戳（由键值对数据结构RedisObject中的lru字段记录）。然后，Redis在决定淘汰的数据时，第一次会随机选出N个数据，把它们作为一个候选集合。接下来，Redis会比较这N个数据的lru字段，把lru字段值最小的数据从缓存中淘汰出去。如果需要淘汰，继续随机挑选N个，选出小于前个N中最小的，加入。  

**LFU，Least Frequently Used(最近最不常使用)** 

核心思想是根据key的最近被访问的频率进行淘汰，很少被访问的优先被淘汰，被访问的多的则被留下来。
#### 5、持久化
Redis的持久化用于保障，机器宕机、重启等场景下，内存数据恢复。Redis是缓存数据库，保持数据的一致性不是Redis的设计初衷。Redis支持2种持久化机制，RDB（内存快照），AOF（命令的追加记录）。
###### RDB
Redis Database，数据快照，把当前进程某一时刻的数据快照保存到硬盘。redis提供两个手动命令支持，save(阻塞主线程)、bgsave(不阻塞主线程)。Redis还支持被动非阻塞出发，在配置文件中配置，save m  n，m秒内数据集存在n次修改。当配置多条是，其中任一满足即可。  

**RDB过程**  

Redis就会借助操作系统提供的写时复制技术（Copy-On-Write,COW），在执行快照的同时，正常处理写操作。简单来说，bgsave子进程是由主线程fork生成的，可以共享主线程的所有内存数据。bgsave子进程运行后，开始读取主线程的内存数据，并把它们写入RDB文件。如果主线程对这些数据也都是读操作（例如图中的键值对A），那么，主线程和bgsave子进程相互不影响。但是，如果主线程要修改一块数据（例如图中的键值对C），那么，这块数据就会被复制一份，生成该数据的副本（键值对C’）。然后，主线程在这个数据副本上进行修改。同时，bgsave子进程可以继续把原来的数据（键值对C）写入RDB文件。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1712048871304-094aae94-6607-4cf6-affc-0126d8f8d980.png#averageHue=%23f4f9e9&clientId=ue5e5e5f3-54ae-4&from=paste&height=3750&id=u506d3f47&originHeight=7500&originWidth=13333&originalType=binary&ratio=2&rotation=0&showTitle=false&size=4448617&status=done&style=none&taskId=ua84bf9cd-6cf4-4873-8bef-feb0eb2a451&title=&width=6666.5)
> 写时复制
> 在实际执行过程中，是子进程复制了主线程的页表，所以通过页表映射，能读到主线程的原始数据，而当有新数据写入或数据修改时，主线程会把新数据或修改后的数据写到一个新的物理内存地址上，并修改主线程自己的页表映射。所以，子进程读到的类似于原始数据的一个副本，而主线程也可以正常进行修改。

**使用RDB持久化，会不会丢数据？**  

一定会丢，Redis本身设计是缓存，不保证需要保证一致性的持久化。bgsave的子进程是通过fork操作从主线程复制出来，fork过程是阻塞主进程的。其次频繁写入磁盘会给磁盘带来比较大的压力。因此不可能会保障内存中的数据和磁盘中完全一致，机器宕机或重启后，根据磁盘RDB恢复，会有新增或变更丢失。
> fork子进程，fork这个瞬间一定是会阻塞主线程的，fork子进程需要拷贝进程必要的数据结构，其中有一项就是拷贝内存页表（虚拟内存和物理内存的映射索引表），这个拷贝过程会消耗大量CPU资源，拷贝完成之前整个进程是会阻塞的，阻塞时间取决于整个实例的内存大小，实例越大，内存页表越大，fork阻塞时间越久。

###### AOF
AOF(append only file)持久化:以独立日志的方式记录每次写命令，Redis先写内存，在把命令写入缓冲后磁盘，这样有个好处，可以避免语法错误，还可以避免阻塞写操作。那缓冲数据什么时候写入磁盘呢？Redis提供三种策略：

- Always：同步写回，可靠性高，数据基本不丢失，性能影响比较大。
- Everysec：每秒写回，性能适中，宕机时丢失1秒内的数据。
- No：操作系统控制，性能好，宕机时丢失数据比较多。

可以根据自己使用场景使用具体的策略。  

**AOF重写机制**  

随着命令越来越多，AOF文件会越来越大，会带来性能问题。主要几个方面，防止日志文件过大，影响到日志追加效率。其次宕机恢复时，日志文件过大，恢复过程就会非常缓慢。
对于同一个键值，AOF会存在多个命令，AOF重写过程，会根据最新状态，生成相应的写入命令。这样就会缩小AOF大小。AOF日志重写过程：主线程fork一个用于bgrewriteaof的子进程，子进程根据主线程的内存数据，都转化为写操作。如果过程中更新，先写入AOF重写缓冲区。重写日志生成后，合并重写缓冲区，后写入磁盘。
##### 混合模式
中提出了一个混合使用AOF日志和内存快照的方法。简单来说，内存快照以一定的频率执行，在两次快照之间，使用AOF日志记录这期间的所有命令操作。这样一来，快照不用很频繁地执行，这就避免了频繁fork对主线程的影响。而且，AOF日志也只用记录两次快照间的操作，也就是说，不需要记录所有操作了，因此，就不会出现文件过大的情况了，也可以避免重写开销。
