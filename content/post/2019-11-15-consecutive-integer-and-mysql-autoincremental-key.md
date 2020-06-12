+++
date = "2019-11-15T00:00:00Z"
tags = ["mysql"]
title = "连续抽奖号码与 MySQL 自增主键"

+++

最近运营要做一个简单的抽奖活动，要求从1开始产生连续的整数，作为抽奖号码发放出去，最后随机抽出一个号码作为幸运号码。一开始没多想，在 mongo 数据库里加了一个 counter, 需要的时候拿出来作为下一个抽奖号码同时原子自增。投入使用后却发现出了问题：产生的整数不连续。

一开始还以为是 mongo 出了问题，细想这是因为中间出现了失败的情况。这种策略太过简单，在失败的情况下 counter 没有回滚，出现间隙几乎是必然的。

这其实跟 MySQL 自增主键的问题非常像。假如 counter 的当前值为 2, 以上的业务过程如下：

1. 获取 counter 的当前值 2, counter 同时原子自增为 3
2. 记录用户 ID 与抽奖号码元组 (user2, 2)
3. 将 (user2, 2) 插入数据库

最后一步插入顺利的话，2 为有效号码，插入失败则 2 被跳过。

对应的，我们初始化一个 MySQL 表。

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;
insert into t values(null, 1, 1);
```

然后插入下面这条数据。

```sql
insert into t values(null, 1, 1);
```

过程如下：

1. 执行器调用 InnoDB 引擎接口写入一行，传入的这一行的值是 (0,1,1);
2. InnoDB 发现用户没有指定自增 id 的值，获取表 t 当前的自增值 2;
3. 将传入的行的值改成 (2,1,1);
4. 将表的自增值改成 3
5. 继续执行插入数据操作，由于已经存在 c=1 的记录，所以报 Duplicate key error，语句返回。

这样，主键值 2 就被跳过了。

mongodb 4.0 以前是没有事务的，就没有所谓回滚的概念。那能不能在业务层回滚下呢？参考MySQL 的实现？但事实上 MySQL 同样没有对自增主键值回滚。

原因在哪呢？假如有两个事务并发如下，transaction1 先获取到 id 2, transaction 3 获取到 id 3, 然后先提交，之后 transaction1 回滚了。

| transaction1 | transaction2 |
|--------------|--------------|
| id = 2       |              |
|              | id = 3       |
|              | commit       |
| rollback     |              |

假设自增主键回滚回 2, 后面再自增一定会跟 transaction2 插入的值冲突。或者以后获取自增主键时一定要确认未被使用，或者必须等一个事务执行完再执行另一个，如下。

| transaction1 | transaction2  |
|--------------|---------------|
| id = 2       |               |
| rollback     |               |
|              | id = 2(not 3) |
|              | commit        |

每新增一条数据，都必须做一次主键查找，或者等待上一次插入完成，无疑极大地降低了写入效率，基本是不可接受的。所以，MySQL 干脆不管了。

用更专业的说法，执行 insert 语句时，自增锁在申请后就马上释放了，并不属于事务的一部分。

在遇到 `insert ... select ...` 语句的时候情况会更加复杂。如果两个 `insert ... select ... ` 语句同时执行，主键值会相互交叉乱序产生。从库执行同样的语句不能保证每条记录获取主键值时与主库一致。所以默认情况下，对于 `insert ... select ...`， MySQL 会等待语句执行完再释放自增锁。为了提升性能，可以设置 `binlog_format=row` 的同时设置 `innodb_autoinc_lock_mode=2`。MySQL 对所有的插入都会马上释放自增锁，用 row binlog_format 保证主从一致。

回到本文最初的问题，因为数据量不大，回滚加主键查找是比较好的方案。当然，最好抛弃 mongo 的原子自增，使用内存数据库。

本文参考了极客时间林晓斌《MySQL 实战 45 讲──自增主键为什么不是连续的》。MySQL 的版本是 8.0.
