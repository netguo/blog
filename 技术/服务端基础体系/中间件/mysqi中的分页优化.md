### 场景
场景源于，最近看到一个sql
```
SELECT A.id, A.uuid, A.report_id, A.monitor_id, A.partner_code, A.app_name, A.person_info, A.risk_detail, A.gmt_create, A.risk_partition, A.extend_field FROM post_loan_risk_tr A, ( SELECT id FROM post_loan_risk_tr WHERE risk_partition = 20191220 AND parner_code=xxx AND app_name=XXX ORDER BY id LIMIT 1000000,100) B WHERE B.id = A.id;
```
risk_partition是mycat的分片键，指定到具体的库。具体用到idx_risk_partition_partner_app索引。索引情况是，（risk_partition,partner_code,app_name)的聚合索引。语句也较为简单，我们想从一个表中根据具体索引进行查询，再做分页。对于这个sql首先有个疑问，为什么需要先选出id？而不是直接执行如下语句。

```
SELECT id, uuid, report_id, monitor_id, partner_code, app_name, person_info, risk_detail, gmt_create, risk_partition, extend_field FROM post_loan_risk_tr  WHERE risk_partition = 20191220  and partner_code= xxx AND app_name = xxx ORDER BY id LIMIT 1000000,100;
```
**mysql中的limit** 

关于这个，我们理解好mysql的limit，就可以了，如下下面代码所示。第二个查询，前offset中的数据也需要进行回表查询。第一个sql中的id为聚集索引，所以只需要查询到id（可以看下面关于覆盖索引的说明），再查询相应的id列就好了。

```
select * from table_name limit 10000,10
这句 SQL 的执行逻辑是
1.从数据表中读取第N条数据添加到数据集中
2.重复第一步直到 N = 10000 + 10
3.根据 offset 抛弃前面 10000 条数
4.返回剩余的 10 条数据
```

**继续explain分析** 

我们看explain结果，第一个图片是未用id子查询的，第一个是使用id子查询的。
![1](../../picture/postloanorderby1.png)
![2](../../picture/postloanorderby2.png)


第二个，用了子查询，我们先看子查询中的查询，与1图中的比较。主要差异点在Extra.
```
Using Index
   表示直接访问索引就能够获取到所需要的数据（覆盖索引），不需要通过索引回表；
   覆盖索引： 如果一个索引包含（或者说覆盖）所有需要查询的字段的值。我们称之为“覆盖索引”。如果索引的叶子节点中已经包含要查询的数据，那么还有什么必要再回表查询呢？
Using where
    表示MySQL服务器在存储引擎收到记录后进行“后过滤”（Post-filter）,如果查询未能使用索引，Using where的作用只是提醒我们MySQL将用where子句来过滤结果集。这个一般发生在MySQL服务器，而不是存储引擎层。一般发生在不能走索引扫描的情况下或者走索引扫描，但是有些查询条件不在索引当中的情况下。

Using Index Condition
     在MySQL 5.6版本后加入的新特性（Index Condition Pushdown）;会先条件过滤索引，过滤完索引后找到所有符合索引条件的数据行，随后用 WHERE 子句中的其他条件去过滤这些数据行；
```

关于using index conditon，可以查看[Index Condition Pushdown](https://www.cnblogs.com/abclife/archive/2017/11/06/7792625.html)，而select id为什么没有使用Index Condition Pushdown，因为id是聚集索引，属于“覆盖索引”，索引extra是using index，using where。也就是此时查询id没有回表操作。

**继续优化orderby**
对于idx_risk_partition_partner_app索引是个b++树，叶子结点是我们聚集索引id，id是自增的，因此无需用继续使用orderby，直接通过limit查找便能保证有序，如果多一个orderby id，会多一次排序动作。
```
SELECT A.id, A.uuid, A.report_id, A.monitor_id, A.partner_code, A.app_name, A.person_info, A.risk_detail, A.gmt_create, A.risk_partition, A.extend_field FROM post_loan_risk_tr A, ( SELECT id FROM post_loan_risk_tr  WHERE risk_partition = 20191220 LIMIT 1000000,1) B WHERE B.id = A.id;
```
我们执行explain可以看到，最终的子查询，只有Using Index，具体是为什么？？？
![3](../../picture/postloanorderby3.png)
