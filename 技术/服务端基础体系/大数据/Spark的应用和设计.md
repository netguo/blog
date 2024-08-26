### 并行计算的发展
21世纪初，互联网行业快速崛起，各个企业数据量逐渐增长，谷歌作为当时全球最大的互联网公司，在海量数据层面遇到巨大的挑战。针对海量数据的处理，谷歌针对行的推出海量数据解决方案，并公开三篇论文，分别是《Google File System》、《Google MapReduce》以及《Google BigTable》。Google 发表了这三篇论文以后，基本上「奠定」了业界大规模分布式存储系统的理论基础。
#### MapReduce模型
MapReduce 架构的程序能够在大量的普通配置的计算机上实现并行化处理。这个系统在运行时只关心: 如何分割输入数据，在大量计算机组成的集群上的调度，集群中计算机的错误处理，管理集群中计算机之间必要的通信。该抽象模型，我们只要表述我们想要执行的简单运算即可，而不必关心并行计算、容错、数据分布、负载均衡等复杂的细节，这些问题都被封装在 了一个库里面。
MapReduce 是一个编程模型，核心概念是将输入数据集映射到健-值对集合，然后对所有包含相同健的健-值对完成归约。MapReduce认为，再复杂的数据处理流程也无非是这两个映射方式的组合。

- 几乎所有的数据都可以映射到健值对。
- 健和值可以是任意类型。
- "不共享"数据处理平台，映射器都可以独立进行，在映射器完后，归约器也能独立工作。

用户自定义的 Map 函数接受一个输入的 key/value pair 值，然后产生一个中间 key/value pair 值的集合。 MapReduce 库把所有具有相同中间 key 值 的中间 value 值集合在一起后传递给 reduce 函数。

- map函数：主节点得到输入，将输入划分为较小的数据库，再将这些数据块分布到工作节点（从节点），工作节点中对各个数据块应用相同的转换函数，然后将结果传回到主节点。
- reduce函数：主节点中根据唯一的健-值对将接收的结果进行洗牌和聚集，然后再一次重新分布到从节点，通过另一类的转换函数组合这些值。
#### Spark的发展
Apache Spark的历史可以追溯到2009年，当时它由加州大学伯克利分校的AMPLab开发。最初，Spark是为了解决Hadoop MapReduce的限制而创建的。随着时间的推移，Spark的生态系统不断壮大，吸引了越来越多的开发者和组织的支持。
目前Spark，具备很多优势，已经成为最流行的分布式计算框架。

- 高性能，通过内存计算来提高性能。
- 多语言支持，包括Scale、Java、Python和R。
- 内置丰富的库，如Spark SQL、MLlib、GraphX等。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1717144575382-cafccfb4-ae1c-4991-8d72-ec6d3c3e04c4.png#averageHue=%23adadad&clientId=u044f6515-01e9-4&from=paste&height=239&id=uf9eaf0ef&originHeight=478&originWidth=810&originalType=binary&ratio=2&rotation=0&showTitle=false&size=110511&status=done&style=none&taskId=u03c12cb3-9652-4256-89af-4fdff84ec9a&title=&width=405)
### Spark的设计和使用
#### Spark架构
![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1717143895402-f95c880d-884f-4ee7-9af3-766cf89f73a5.png#averageHue=%23f2eeeb&clientId=u044f6515-01e9-4&from=paste&height=143&id=Pi3gM&originHeight=286&originWidth=596&originalType=binary&ratio=2&rotation=0&showTitle=false&size=29704&status=done&style=none&taskId=u99e23009-909a-4a3e-bdee-f119bee2aef&title=&width=298)
Spark采用的是主从架构，有Driver进行任务划分和调度，Cluster Manager管理机器的资源，Driver通过Cluster Manager把任务分配个不同的Worker。
首Driver根据用户编写的代码生成一个计算任务的有向无环图（Directed Acyclic Graph, DAG），接着，DAG会根据RDD（弹性分布式数据集）之间的依赖关系被DAGScheduler切分成不同的Stage，每个Stage对应一组Task，一个Task对应一个Partition，整体是多组TaskSet, TaskScheduler会通过ClusterManager将任务调度到Executor上执行。一个Executor同时只能执行一个Task，但一个Worker（物理节点）上可以同时运行多个Executor。
在MapReduce这类型的计算框架中，中间结果的传输是整个计算过程中最重要的一个步骤，Spark也是如此，在Spark作业中，这也是Stage划分的依据，我们称之为数据混洗（Shuffle）。
#### RDD
RDD是Spark分布式计算引擎的基石，很多Spark的核心概念和核心组建，如DAG和调度系统都衍生自RDD。尽管现在RDD API使用频率越来越低，但是Data Frame和DataSet API在Spark内部最终都会转化为RDD做分布式计算。RDD 是一种抽象，是 Spark 对于分布式数据集的抽象，它用于囊括所有内存中和磁盘中的分布式数据实体。
RDD的数据的4个属性

- partitions：数据分片
- partitioner：分片切割规则
- dependencies：RDD依赖
- compute：转换函数

RDD通过SparkContext创建，可以内部定义的数据创建，也可以通过通过外部数据文件、Hive、Hbase等创建。
RDD支持两种类型的操作: transformations（转换）和actions（动作），转换函数会形成一个新的RDD，动作函数将在RDD上运行计算结果返回到driver程序。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1717383200793-d7b5d921-429a-4dd2-b50d-e4c5af85b215.png#averageHue=%23c5d5e9&clientId=u839a9584-89b8-4&from=paste&height=1366&id=uf4d24439&originHeight=1366&originWidth=1880&originalType=binary&ratio=2&rotation=0&showTitle=false&size=281324&status=done&style=none&taskId=u0abd7254-b4a5-4867-91c2-488e6a793f9&title=&width=1880)
#### 常用的算子
**转化函数**
_map_
给定映射函数f，map(func)以元素为粒度对RDD所有数据转换，产生一个新的分布式数据集。func可以是具体一个函数，也可以是匿名函数。
_mapPartitions_
函数的形参是以partition为力度的，函数内部还可以以partition粒度进一步使用转换函数。可以在partition转换之前，建立公共的实例，例如数据库链接等。
_flatmap_
跟map类似，可以平铺数据。可以理解为，对于每一个数据产生一个集合，最后把所有集合合并的过程。
```
rdd.flatMap(_.split(" ")).collect
最终产生，每一行通过" "划分后的所有元素。
```
**聚合函数**
聚合函数只能作用于Paired RDD上，指的是元素类型为(Key,Value)的键值对的RDD。聚合函数都会产生shutter。常用的聚合函数如下：
_groupByKey_
分组收集，根据key为维度进行分组收集，相同的Key的Value形成一个集合。
_reduceByKey_
分组聚合，可以带聚合函数，聚合函数func的类型必须是（Value类型,Value类型）=>(Value类型)
```
rdd中元素类型（String，Int），最终返回，（Key，Value的和）
rdd.reduceBykey((x,y)=>x+y)
```
相比 groupByKey 以全量原始数据记录的方式消耗磁盘与网络，reduceByKey 在落盘与分发之前，会先在 Shuffle 的 Map 阶段做初步的聚合计算。
_aggregateByKey_
aggregateByKey提供两个函数，Map端的聚合函数f1，以及Reduce端聚合函数f2。例如先求和，再取最大值。f1的形参类型，必须与Paired RDD的value类型保持一致，f2的形参，必须与f1的的结果保持一致。
_sortByKey_
根据key做生序、降序
**数据重分布**
repartition算子随意调整（提升或降低）RDD的并行度，而coalesce算子则只能用于降低RDD并行度。
repartition的计算过程都是先哈希、再取模，得到的结果便是该条数据的目标分区索引。对于绝大多数的数据记录，目标分区往往坐落在另一个 Executor、甚至是另一个节点之上，因此 Shuffle 自然也就不可避免。coalesce 则不然，在降低并行度的计算中，它采取的思路是把同一个 Executor 内的不同数据分区进行合并，如此一来，数据并不需要跨 Executors、跨节点进行分发，因而不会引入 Shuffle。
**结果收集**
first 用于收集 RDD 数据集中的任意一条数据记录，而 take(n: Int) 则用于收集多条记录。collect 拿到全量数据，也就是把 RDD 的计算结果全量地收集到 Driver 端。collect 算子有两处性能隐患，一个是拉取数据过程中引入的网络开销，另一个 Driver 的 OOM（内存溢出，Out of Memory）。collect 算子有两处性能隐患，一个是拉取数据过程中引入的网络开销，另一个 Driver 的 OOM（内存溢出，Out of Memory）。
持久化到磁盘，最具代表性的非 saveAsTextFile 莫属，它的用法非常简单，给定RDD，我们直接调用 saveAsTextFile(path: String) 即可。其中 path 代表的是目标文件系统目录，它可以是本地文件系统，也可以是 HDFS、Amazon S3 等分布式文件系统。
####  shuffle
在 MapReduce 框架中， Shuffle 阶段是连接 Map 与 Reduce 之间的桥梁， Map 阶段通过 Shuffle 过程将数据输出到 Reduce 阶段中。由于 Shuffle 涉及磁盘的读写和网络 I/O，因此 Shuffle 性能的高低直接影响整个程序的性能。
shuffle分为两个阶段，Shuffle Write和Shuffle Read，第一阶段是Map Task写文件，第二阶段是Reduce Task读文件。Shuffle Write过程，Map Task需要根据Reduce的 Task个数进行分片写入。
**HashShuffleManager**
每个Map Task为每一个Reduce Task生成一个文件，总共生成M*R个文件。多个Map，会生成N*M*R个小文件，造成大量的文件碎片和磁盘压力。后来提出了shuffleFileGroup概念，不同阶段的Map Task可以共用文件。Spark2.0以后已经默认不采用该模式。
**SortShuffleManager**

- 普通运行模式：每一个Map Task只生成一个数据文件，还有一个索引文件标记每个Reduce的起始位置。实现方式，通过内存缓冲写文件，形成多个小文件，最终做归并排序生成一个大文件。简易流程，每一行数据先哈希取模形成RID，并写入缓存，再写入文件，归并排序根据RID+KEY
- bypass模式：实现了一个类似Hash风格的回退方案，先写多个小文件，最终合并成一个大文件，并创建索引文件。毕竟排序成本也高，采用两个参数限定，map task小于某值，默认200，且不是聚合类shuffle算子。
- Tungsten-Sort：一种排序算法的改进，通过二进制排序，减少序列化。
#### 内存管理
Spark内存分为4个区域，Reserver Memory、User Memory、Execution Memory、Storage Memory

- Reserved Memory 固定为 300MB，不受开发者控制，它是 Spark 预留的、用来存储各种 Spark 内部对象的内存区域。
- User Memory 用于存储开发者自定义的数据结构，例如 RDD 算子中引用的数组、列表、映射等等。
- Execution Memory 用来执行分布式任务。
- Storage Memory 用于缓存分布式数据集，比如 RDD Cache、广播变量等等。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1717575305653-49ac0908-db87-4abf-8c4e-49d53b3eb0e7.png#averageHue=%23f2f4ef&clientId=uad1aec64-d867-4&from=paste&height=143&id=u83f32e6d&originHeight=286&originWidth=976&originalType=binary&ratio=2&rotation=0&showTitle=false&size=45419&status=done&style=none&taskId=uc7819239-60dd-4177-ac52-e5531948242&title=&width=488)
在所有的内存区域中，Execution Memory 和 Storage Memory 是最重要的，也是开发者最需要关注的。在 Spark 1.6 版本之前，Execution Memory 和 Storage Memory 的空间划分是静态的，一旦空间划分完毕，不同内存区域的用途与尺寸就固定了。现在如果对方的内存空间有空闲，可以相互占用。
缓存
### Spark SQL
#### 为什么需要Spark SQL
RDD提供的函数，像map,flatmap等都是高阶函数，形参都是函数。开发者可以灵活的进行函数撰写，那么这样会带来什么问题呢？执行器Spark Core很难知道开发者要“做什么”，那么也很难有优化空间。
Spark社区在1.3版本发布了DataFrame，与RDD一样都是分布式数据集。它与RDD的区别：

- DataFrame是携带数据模式的结构化数据
- DataFrame自定义了一套DSL算子

那么优化空间打开之后，真正负责优化内核的是Spark SQL，它是在Spark Core之上的，Spark SQL代码优化后，还是交给Spark Core执行。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1718098909260-06a972ca-e3a8-42d7-8915-8059b5fb4613.png#averageHue=%23d1e3c3&clientId=u540ed9a0-ad14-4&from=paste&height=110&id=u6d2e50c1&originHeight=220&originWidth=938&originalType=binary&ratio=2&rotation=0&showTitle=false&size=35340&status=done&style=none&taskId=u10ac532d-ae66-4b35-aa1d-86360d45ea3&title=&width=469)
#### 优化器
基于DataFrame，Spark SQL主要两个优化组件：Catalyst优化器和Tungsten。Catalyst主要职责在于创建并优化执行计划，它包含三个功能模块：创建语法树并生产执行计划、逻辑阶段优化和物理阶段优化。Tungsten用于衔接Catalyst执行计划和底层的Spark Core执行引擎。
**Catalyst优化器**
Catalyst会先使用第三方的SQL解析器ANTLR生成抽象语法树AST，逻辑优化阶段，基于启发式的规则和策略调整，例如Filter提前。物理优化阶段更多是通过数据进行优化，例如Join节点，通过两张表的数据大小，选定Join方式。
**Tungsten优化器**
Tungsten 设计并实现了一种叫做 Unsafe Row 的二进制数据结构。Unsafe Row 本质上是字节数组，它以极其紧凑的格式来存储 DataFrame 的每一条数据记录，大幅削减存储开销，从而提升数据的存储与访问效率。
Tungsten在运行时把算子之间的“链式调用”捏合为一份代码。这样一来，Spark Core 只需要使用函数来一次性地处理每一条数据，就能消除不同算子之间数据通信的开销，一气呵成地完成计算。
#### DataFrame的使用
Spark支持多种数据源，从类别划分：自定义数据结果，文件系统，关系型数据库，数据仓库，NoSQL，其它引擎(例如Kafka)。
DataFrame常用创建方式，通过RDD创建，RDD的类型必须是RDD[Row]，手工指定Data Schema。也可以通过RDD的隐士转化方法toDF。像是关系性数据库、数仓等自带Scheme，可以通过format、load数据加载。
Spark SQL，采用常用的SQL语言，可以通过Spark Session直接执行，此外还支持丰富的内置函数。
**DataFrame算子**
![image.png](https://cdn.nlark.com/yuque/0/2024/png/8364057/1718177322051-6a4a2c54-94fb-4dc1-8474-93436bf01390.png#averageHue=%23f4e5cf&clientId=u772ca6c3-85d3-4&from=paste&height=145&id=udd404d26&originHeight=289&originWidth=1575&originalType=binary&ratio=2&rotation=0&showTitle=false&size=58657&status=done&style=none&taskId=u01e7b4c4-9246-4f1d-8cb4-b77a3f80431&title=&width=787.5)
#### 数据关联
Spark SQL支持的关联方式种类非常丰富，按照关联方式可以分为：内关联、外关联、左关联、右关联、左半关联、左逆关联。左半，左逆都是取子集，左半可以选取其中几列，左逆选取几列外的列。
Join有三种实现方式：NLJ，SML，HJ

- NLJ：驱动表扫描基表，O(M*N)
- SML：先排序，2个表再归并。O(logM)，表有序的情况下O(M + N)
- HJ：先对每个表建立Hash，再根据驱动表，hash查找基表。O(M)，需要额外存储空间。

从分发模式，Spark支持Shuffle、Broadcast，结合上述三种实现方式，总共会有6种选择。
Spark SQL的选择：

- 等值关联，首先考虑Broadcast+HJ，条件是被广播的表满足内存条件，根据内存设置而定
- 等值关联，如果不满足Broadcast条件，首先考虑Shuffle+SML，因为在map过程，默认的生成的文件是排序的。执行效率跟HJ差不多，且不需要额外的内存。
- 非等值关联，只能通过NLJ，如果内存满足选择Broadcast。


