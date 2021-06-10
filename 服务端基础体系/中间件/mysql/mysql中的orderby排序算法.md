
一 简介

偏向于业务的 (MySQL)DBA 或者业务的开发者来说，order by 排序是一个常见的业务功能，将结果根据指定的字段排序，满足前端展示的需求。然而排序操作也是经常出现慢查询排行榜的座上宾。本文将从原理和实际案例优化，order by 使用限制等几个方面来逐步了解 order by 排序。

二 原理

在了解 order by 排序的原理之前，强烈安利两篇关于排序算法的文章 《归并排序的实现》 《经典排序算法》。MySQL 支持两种排序算法，常规排序和优化，并且在 MySQL 5.6 版本中 针对 order by limit M,N 做了特别的优化，这里列为第三种。

2.1 常规排序

a 从表 t1 中获取满足 WHERE 条件的记录

b 对于每条记录，将记录的主键 + 排序键 (id,col2) 取出放入 sort buffer

c 如果 sort buffer 可以存放所有满足条件的 (id,col2) 对，则进行排序;否则 sort buffer 满后，进行排序并固化到临时文件中。(排序算法采用的是快速排序算法)

d 若排序中产生了临时文件，需要利用归并排序算法，保证临时文件中记录是有序的

e 循环执行上述过程，直到所有满足条件的记录全部参与排序

f 扫描排好序的 (id,col2) 对，并利用 id 去捞取 SELECT 需要返回的列 (col1,col2,col3)

g 将获取的结果集返回给用户。

从上述流程来看，是否使用文件排序主要看 sort buffer 是否能容下需要排序的 (id,col2) 对，这个 buffer 的大小由 sort_buffer_size 参数控制。此外一次排序需要两次 IO，一次是捞 (id,col2), 第二次是捞 (col1,col2,col3)，由于返回的结果集是按 col2 排序，因此 id 是乱序的，通过乱序的 id 去捞 (col1,col2,col3) 时会产生大量的随机 IO。对于第二次 MySQL 本身一个优化，即在捞之前首先将 id 排序，并放入缓冲区，这个缓存区大小由参数 read_rnd_buffer_size 控制，然后有序去捞记录，将随机 IO 转为顺序 IO。

2.2 优化排序

常规排序方式除了排序本身，还需要额外两次 IO。优化的排序方式相对于常规排序，减少了第二次 IO。主要区别在于，放入 sort buffer 不是 (id,col2), 而是 (col1,col2,col3)。由于 sort buffer 中包含了查询需要的所有字段，因此排序完成后可以直接返回，无需二次捞数据。这种方式的代价在于，同样大小的 sort buffer，能存放的 (col1,col2,col3) 数目要小于 (id,col2)，如果 sort buffer 不够大，可能导致需要写临时文件，造成额外的 IO。当然 MySQL 提供了参数 max_length_for_sort_data，只有当排序元组小于 max_length_for_sort_data 时，才能利用优化排序方式，否则只能用常规排序方式。

2.3 优先队列排序

为了得到最终的排序结果，无论怎样，我们都需要将所有满足条件的记录进行排序才能返回。那么相对于优化排序方式，是否还有优化空间呢?5.6 版本针对 Order by limit M，N 语句，在空间层面做了优化，加入了一种新的排序方式: 优先队列，这种方式采用堆排序实现。堆排序算法特征正好可以解 limit M，N 这类排序的问题，虽然仍然需要所有元素参与排序，但是只需要 M+N 个元组的 sort buffer 空间即可，对于 M，N 很小的场景，基本不会因为 sort buffer 不够而导致需要临时文件进行归并排序的问题。对于升序，采用大顶堆，最终堆中的元素组成了最小的 N 个元素，对于降序，采用小顶堆，最终堆中的元素组成了***的 N 的元素。

三 优化

通过上面的原理分析，我们知道排序的本质是通过一定的算法 (耗费 cpu 运算, 内存, 临时文件 IO) 将结果集变成有序的结果集。如何优化呢?答案是分两个方面利用索引的有序性 (MySQL 的 B+ 树索引是默认从小到大递增排序) 减少排序, ***的方式是直接不排序。

create table t1( 
  id int not null primary key , 
  key_part1 int(10) not null, 
  key_part2 varchar(10) not null default '', 
  key_part3 
  key idx_kp1_kp2(key_part1,key_part2,key_part4)， 
  key idx_kp3(id) 
) engine=innodb default charset=utf8 
以下种类的查询是可以利用到索引 idx_kp1_kp2 的

SELECT * FROM t1 ORDER BY key_part1,key_part2,... ; 
SELECT * FROM t1 WHERE key_part1 = constant ORDER BY key_part2; 
SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 DESC; 
SELECT * FROM t1 WHERE key_part1 = 1 ORDER BY key_part1 DESC, key_part2 DESC; 
SELECT * FROM t1 WHERE key_part1 > constant ORDER BY key_part1 ASC; 
SELECT * FROM t1 WHERE key_part1 < constant ORDER BY key_part1 DESC; 
SELECT * FROM t1 WHERE key_part1 = constant1 AND key_part2 > constant2 ORDER BY key_part2 
温馨提示 ，各位看官要辩证的看待官方给的例子，自己多动手实践。

无法利用到索引排序的情况，其实我觉得这是本文的重点，对于广大开发同学而言，记住那种不能利用索引排序会更简单些。

1. 最常见的情况 用来查找结果的索引 (key2) 和 排序的索引 (key1) 不一样,where a=x and b=y order by id;

SELECT * FROM t1 WHERE key2=constant ORDER BY key1; 
2. 排序字段在不同的索引中，无法使用索引排序

SELECT * FROM t1 ORDER BY key1,key2; 
3. 排序字段顺序与索引中列顺序不一致，无法使用索引排序，比如索引是 key idx_kp1_kp2(key_part1,key_part2)

SELECT * FROM t1 ORDER BY key_part2, key_part1; 
4. order by 中的升降序和索引中的默认升降不一致，无法使用索引排序

SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC; 
5. ey_part1 是范围查询，key_part2 无法使用索引排序

SELECT * FROM t1 WHERE key_part1> constant ORDER BY key_part2; 
5 rder by 和 group by 字段列不一致

SELECT * FROM t1 WHERE key_part1=constant ORDER BY key_part2 group by key_part4; 
6. 索引本身是无序存储的，比如 hash 索引，不能利用索引的有序性。

7. order by 字段只被索引了前缀 ，key idx_col(col(10))

select * from t1 order by col ; 
8. 对于还有 join 的关联查询，排序字段并非全部来自于***个表，使用 explain 查看执行计划***个表 type 值不是 const 。

当无法避免排序操作时, 又该如何来优化呢?很显然, 优先选择 using index 的排序方式，在无法满足利用索引排序的情况下，尽可能让 MySQL 选择使用第二种单路算法来进行排序。这样可以减少大量的随机 IO 操作, 很大幅度地提高排序的效率。

1. 加大 max_length_for_sort_data 参数的设置

在 MySQL 中, 决定使用老式排序算法还是改进版排序算法是通过参数 max_length_for_sort_data 来决定的。当所有返回字段的***长度小于这个参数值时, MySQL 就会选择改进后的排序算法, 反之, 则选择老式的算法。所以, 如果有充足的内存让 MySQL 存放须要返回的非排序字段, 就可以加大这个参数的值来让 MySQL 选择使用改进版的排序算法。

2. 去掉不必要的返回字段

当内存不是很充裕时, 不能简单地通过强行加大上面的参数来强迫 MySQL 去使用改进版的排序算法, 否则可能会造成 MySQL 不得不将数据分成很多段, 然后进行排序, 这样可能会得不偿失。此时就须要去掉不必要的返回字段, 让返回结果长度适应 max_length_for_sort_data 参数的限制。

同时也要规范 MySQL 开发规范，尽量避免大字段。当有 select 查询列含有大字段 blob 或者 text 的时候, MySQL 会选择常规排序。

" algorithm to use. It normally uses the modified algorithm except when or columns are involved, in which case it uses the original algorithm."

3. 增大 sort_buffer_size 参数设置

这个值如果过小的话, 再加上你一次返回的条数过多, 那么很可能就会分很多次进行排序, 然后***将每次的排序结果再串联起来, 这样就会更慢, 增大 sort_buffer_size 并不是为了让 MySQL 选择改进版的排序算法, 而是为了让 MySQL 尽量减少在排序过程中对须要排序的数据进行分段, 因为分段会造成 MySQL 不得不使用临时表来进行交换排序。但是这个值不是越大越好：

1. sort_buffer_size 是一个 connection 级参数, 在每个 connection ***次需要使用这个 buffer 的时候, 一次性分配设置的内存。

2. sort_buffer_size 并不是越大越好, 由于是 connection 级的参数, 过大的设置 + 高并发可能会耗尽系统内存资源。

3. 据说 sort_buffer_size 超过 2M 的时候, 就会使用 mmap() 而不是 malloc() 来进行内存分配, 导致效率降低。