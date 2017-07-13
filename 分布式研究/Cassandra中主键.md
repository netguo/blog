
#### 关系型数据库中主键
关系数据库中的主键可以理解为唯一性索引，是由一列或者几列组成。MySQL中的主键一般建议为非业务性的自增字段。  

#### Cassandra中的主键
与关系型数据库不同，Cassandra的存储结构决定，Cassandra中必须存在主键。
Partition Key ：决定了数据在Cassandra各个节点的是如何分区的。
Clustering Key ： 用于在各个分区内的排序。
Primary Key ： 主键，决定数据行的唯一性。
Composite Key ：只是一个多字段组合的概念。

