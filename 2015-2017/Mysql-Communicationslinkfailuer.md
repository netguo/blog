最近几个业务线在MySQL查询时，报了一下错误。
java.lang.RuntimeException: org.springframework.dao.RecoverableDataAccessException: 
### Error querying database.  Cause: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet successfully received from the server was 5,005 milliseconds ago.  The last packet sent successfully to the server was 5,005 milliseconds ago.

触发这个异常的原因是，查询时间较长，大部分操作是count(*) 引起的

解决方案：
1. 暴力方式：
延长查询时间，现在我们MySQL的配置中未配置queryTimeOut和transactionQueryTimeout，但是配置了socketTimeout，和connectTimeout，可以对这个做个延长，原先是5000ms，可以变更为30000ms

2. 优化SQL
加索引等优化
特殊场景：
对于count(*)的操作，在一些业务上，如果数据量太大，即便有索引也很难比较查询时间较长，建议与产品协商展示的最大数量，对count(*)也做limit操作，经测试10W数据大概在2S内。

示例：select count(*) from (select id from workflow_history where  .... order by .. limit 100000) 
