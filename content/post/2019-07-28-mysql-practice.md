---
date: "2019-07-28T00:00:00Z"
tags:
- mysql
title: 《MySQL 实战 45 讲》拾遗
---

极客时间上林晓斌的《MySQL 实战 45 讲》针对日常开发碰到的常见问题都有非常高的实践指导意义，虽然说在这一系列文章也简述了一些数据库原理，但要把整个系列的文章消化下来还是要具备相当的数据库基础。我是在看过下面几本书才看这个系列文章的，一来使得理解起文章中的例子相对轻松，二来有了具体的可操作实例，对照起书中的原理，也经常有恍然大明白的感觉。

- 罗摩克里希纳. 数据库管理系统原理与设计. 清华大学出版社, 2003. http://book.douban.com/subject/1146233/.
- 施瓦茨, 扎伊采夫, and 特卡琴科. 高性能MySQL. Translated by 宁海元, 周振兴, 彭立勋, and 翟卫祥刘辉. 电子工业出版社, 2013. https://book.douban.com/subject/23008813/.
- 姜承尧. MySQL技术内幕. 机械工业出版社, 2010. https://book.douban.com/subject/5373022/.
- MICK, ［ 日］. SQL进阶教程. Translated by 吴炎昌. 人民邮电出版社, 2017. https://book.douban.com/subject/27194738/.

下面并不是非常详细的笔记，仅仅是记录个人在学习中认为需要注意的一些点。

## 02 日志系统

`innodb_flush_log_at_trx_commit` 为 1 时每次完成事务后 redo log 写到磁盘。

`sync_binlog` 为 1 时每次完成事务后 binlog 写到磁盘。

## 03 事务隔离
可以用下面的语句查找持续时间超过 60s 的事务。
```sql
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

## 06 全局锁和表锁
在可重复读隔离等级下用 `mysqldump -single-transaction` 可以得到一致性的数据备份。

innodb 的表级锁包括表锁和元数据锁(meta data lock, MDL), 元数据锁包括读锁和写锁。对表做增删改查时加读锁，对表结构进行修改时加写锁。

## 07 行锁功过
TODO 绘制 mysql 单行自增写速率曲线

## 08 事务到底是隔离的还是不隔离的

undo log 指的是数据的事务快照和当前版本的差值，注意不要和 redo log 记混名字。

## 09 普通索引和唯一索引，应该怎么选择？

redo log 节省了把随机写转成了顺序写，change buffer 起的是数据缓存的作用，节省了随机读磁盘的 IO 消耗。

## 10 MySQL 为什么会选错索引
文中的第一个案例一直复现不出来。

## 12 为什么我的 MySQL 会抖一下
设置好 `innodb_io_capacity` 等于实际硬盘性能很重要。可以用 `fio` 测量。

控制脏页比例，可以从全局状态获取 `Innodb_buffer_pool_pages_dirty` 和 `Innodb_buffer_pool_pages_total`.

`innodb_flush_neighbors` 不适用于 SSD。

## 13 为什么表数据删掉一半，表文件大小不变
```sql
ALTER TABLE t ENGINE = InnoDB;
```
可以用于重建表，压缩空间。

`gh-ost`, aka, github online schema migration for mysql. 好东西，实现原理是使用 binlog 而不是 trigger.

## 14 count(*) 这么慢，我该怎么办
由于使用 MVCC, innodb 会取出数据进行计数。优化器会进行一定的优化，比如使用较小的索引树。

要快速得到精确的 count 只能另外存储。用 redis 会有分布式一致性问题。可以在 mysql 另表存储，解决一致性问题。

速度上，count(字段) < count(主键) < count(1) ≈ count(\*). count(\*) 有优化过的语义，表示统计行数。其它都是按字面意思执行。

## 16 "order by" 是怎么工作的
这篇文章介绍了optimizer_trace, 比起 explain，能得到更详细的信息。

```sql
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a 保存 Innodb_rows_read 的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b 保存 Innodb_rows_read 的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算 Innodb_rows_read 差值 */
select @b-@a;
```

`max_length_for_sort_data` 控制 MySQL 排序时行数据的长度，单行长度小于该值，使用全字段排序，大于该值，使用 rowid 排序。

## 17 如何正确地显示随机消息
在 `sort_buffer_size` 足够大时，mysql 会优先使用优先队列处理 `SELECT ... LIMIT...` 类型的语句。

## 19 为什么我只查一行的语句，也执行这么慢
查询数据量很少，但是却很慢的问题诊断过程。

先找出问题线程及其相关信息
```sql
show processlist
```

查找表相关信息
```sql
select * from sys.innodb_lock_waits where locked_table = t\G
```

查找搜索时间较长的语句。
```sql
set long_query_time=0
```

## 21 为什么我只改一行的语句，锁这么多
首先要说的是本章的规则和案例都非常珍贵，正如作者所说，可能在别的地方都找不到类似的总结。然而认真看完之后我却一改以前优先使用可重复读的观点了。在现实中，开发人员水平实在是太过参差不齐。即使是优秀的程序员也会在开发的时候顾此失彼。在缺少专业 DBA 的情况下，尽可能使用提交读。遇到无法克服的问题才使用可重复读。

可重复读加锁经验规则：

```
原则1: 加锁的过程的基本单位是 next-key lock.

原则2: 查找过程中访问到的对象才会加锁。

优化1: 索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。

优化2: 索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙。

bug: 唯一索引上的范围查询会访问到不满足条件的第一个值为止。
```

# 23 MySQL 是怎么保证数据不丢的

出现 IO 性能瓶颈时，可以将 `sync_binlog` 设成 N(常见100~1000). 设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count` 通过拉长响应时间来提高吞吐量，原理是组提交策略。

# 24-29 多库架构
binlog 至少是 mixed 才能保证主备一致。

这是一系列多库架构的文章，因为自己缺乏多库的经验，经常遇到读不懂。

TODO 手动搭建主备分离结构。

# 30 用动态的观点看加锁
怎么查看死锁，下面的语句可以查到最后检测到的死锁
```sql
show engine innodb status
```

## others

```sql
show profiles
```

可以临时打开 profiling, 高精度地记录每个步骤的执行时间。

```sql
SET global profiling = ON;
```

# 33 我查这么多数据，会不会把数据库内存打爆
innodb 的 buffer pool 管理使用改进的 LRU 算法，以对付类似全表扫描的极端情况。
# 35 join 语句怎么优化
基本的 JOIN 算法有两种：Index Nested-Loop Join(NLJ) 和 Block Nested-Loop Join(BNL).

大表 join 操作对 buffer pool 即对内存命中率的影响是持续性的，innodb 改进过的 LRU 无法有效地应对这种情况。这时最基本的优化是在被驱动表上建立索引，以使用 BKA(batched key access) 算法进行 JOIN 操作。

启用 BKA 之前需要设置：
```sql
set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
```

MRR: multi-range read

当遇到不适合加索引的情况，可以使用带有索引的临时表，插入大表筛选过的数据，以此使用 BKA 算法。

在目前 mysql 不支持 hash-join 的情况下，可以自己在业务层做一定程度的模拟，可能可以获得比临时表更佳的效率。

# 36 为什么临时表可以重名

`create temporary table` 可以创建临时表，只对当前 session 可见，与普通表同名时优先使用，但是 `show tables` 命令不显示临时表。临时表特别适用于 JOIN 优化。

在 binlog_format='row' 时临时表记录不记录到 binlog. 可以省去不少麻烦。

# 37 什么时候会使用内部临时表
`group by` 默认排序，加上 `order by null` 可以跳过排序。

善加利用 `generated column` 加索引来优化查询。如下面查询可以增加 `id%100` generated column

```sql
select id%100 as m, count(*) as c from t1 group by m;
```

对于 `group by`，还可以用 `SQL_BIG_RESULT` 作为 hint 使用数组(sort buffer)而不是临时表来存储，比如下面的查询。

```sql
select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;
```

# 38 都说 InnoDB 好，那还要不要使用 Memory 引擎
InnoDB 是索引组织表，使用 B+ 树索引，Memory 引擎是堆组织表，使用哈希索引。

除了临时表，都应该使用 Innodb 代替内存引擎。

# 39 自增主键为什么不是连续的
自增主键不连续主要有以下三种原因

1. 唯一键冲突导致插入失败
2. 事务回滚
3. 批量插入预申请过多 ID.

默认设置下，`innodb_autoinc_lock_mode` 值为 1, 表示对于普通插入语句，自增锁申请后马上释放，对于批量插入语句，自增锁需要等语句结束后才释放，以防主备不一致。
