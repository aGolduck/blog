+++
date = "2020-05-18T00:00:00Z"
tags = ["database", "ORM"]
title = "数据库自动迁移"

+++

在小型应用以及应用原型快速开发阶段，关系数据库表定义
自动迁移是非常方便的特性。现在成熟的 ORM 都有所支持。
以 typorm 为例，一般来说，我们进行一次成功的数据库迁移需要包括下面几步。

1. 比较代码与数据库的表头定义，生成迁移脚本 `typeorm migration:generate -n`
2. 编辑迁移脚本，使迁移过程更合理，更数据安全，有必要时可生成空脚本编辑 `typeorm migration:create`
3. 运行迁移脚本，原理一般是数据库有一个迁移元数据表，记录已执行的脚本 `typeorm migration:run`
4. 有必要的情况下回滚 `typeorm migration:revert`


typeorm 对数据库自动迁移有非常好的支持。所有与迁移相关的选项，
与其它 typeorm 配置选项都可通过环境变量来直接配置。

```
TYPEORM_ENTITIES=dist/**/*.entity.js    ## 表头定义代码文件
TYPEORM_SYNCHRONIZE=false     ## 设置为 false, typeorm 才不会每次启动清空数据库
TYPEORM_MIGRATIONS_RUN=true    ## 设置为 true, typeorm 在每次启动项目时都会自动执行迁移，无需手动
TYPEORM_MIGRATIONS=dist/migrations/**/*.js    ## 编译成功迁移脚本，用于迁移
TYPEORM_MIGRATIONS_DIR=src/migrations    ## typeorm 生成的 ts 文件放置的位置
```

TYPEORM_MIGRATIONS_RUN 提供了每次启动自动迁移的选项。
如果是单机布署的话，十分方便，每次上线后可自动完成迁移工作。
对于多机布署，本来就需要轮留更新使得应用保持一贯性。
所以只要布署过程得当，一样可以做自动迁移的工作。

[https://dev.mysql.com/doc/refman/8.0/en/atomic-ddl.html](mysql 文档)
明确说了：

> Atomic DDL is not transactional DDL. DDL statements, atomic or otherwise, implicitly end any transaction that is active in the current session, as if you had done a COMMIT before executing the statement. This means that DDL statements cannot be performed within another transaction, within transaction control statements such as START TRANSACTION ... COMMIT, or combined with other statements within the same transaction.

任何的 DDL(数据定义语句)执行前都会自动作一个 `commit` 的动作. GAME OVER!
流行的果然是辣鸡！我们来看看世界上最先进的开源数据库。

![postgresql]({{site.url}}/assets/google-postgresql.jpg)

果然是最先进的开源数据库，请进入

https://wiki.postgresql.org/wiki/Transactional_DDL_in_PostgreSQL:_A_Competitive_Analysis

看 pg 吊打一众主流数据库，尤其是 mysql.

```
$ psql mydb
mydb=# DROP TABLE IF EXISTS foo;
NOTICE: table "foo" does not exist
DROP TABLE
mydb=# BEGIN;
BEGIN
mydb=# CREATE TABLE foo (bar int);
CREATE TABLE
mydb=# INSERT INTO foo VALUES (1);
INSERT 0 1
mydb=# ROLLBACK;
ROLLBACK
mydb=# SELECT * FROM foo;
ERROR: relation "foo" does not exist
mydb=# SELECT version();
version
----------------------------------------------------------------------
PostgreSQL 8.3.7 on i386-redhat-linux-gnu, compiled by GCC gcc (GCC) 4.3.2 20081105 (Red Hat 4.3.2-7)
(1 row)
```

```
mysql> drop table if exists foo;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> create table foo (bar int) type=InnoDB;
Query OK, 0 rows affected, 1 warning (0.03 sec)

mysql> insert into foo values (1);
Query OK, 1 row affected (0.00 sec)

mysql> rollback;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from foo;
+------+
| bar |
+------+
| 1 |
+------+
1 row in set (0.00 sec)

mysql> select version();
+--------------------------+
| version() |
+--------------------------+
| 5.0.32-Debian_7etch1-log |
+--------------------------+
1 row in set (0.00 sec)
```

2016 年有一篇很有影响的 uber 的为什么放弃 pg 的文章。
他们最终的选择是在 mysql 上加一层。具体可见
https://eng.uber.com/postgres-to-mysql-migration/

国内广泛地使用 mysql, 主要是历史惯性，积攒下了大量的优秀 mysql
运维人才。近几种版本 mysql 的效率和特性都有很大改善。

在应用原型阶段和中小应用，pg 仍不失为一种优秀的选择。
至多用 ORM 保持数据库独立，需要的时候做次迁移就行了。
