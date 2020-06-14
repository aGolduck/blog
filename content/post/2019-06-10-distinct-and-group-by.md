+++
date = "2019-06-10T00:00:00Z"
tags = ["mysql"]
categories = ["database"]
title = "Distinct and Group By"

+++

在 MySQL 里面, `DISTINCT` 原则上可以用 `GROUP BY` 替换。

比如以下的 `question` 表是我们实际生产数据一个表的简化。

```sql
SHOW FULL COLUMNS FROM question;
```

| Field | Type                                                                                      |
|----- |----------------------------------------------------------------------------------------- |
| id    | int(10) unsigned                                                                          |
| type  | enum('choice','single\_choice','uncertain\_choice','determine','fill','essay','material') |
| score | float(10,1) unsigned                                                                      |

```sql
SHOW INDEX FROM question;
```

| Table    | Non\_unique | Key\_name | Seq\_in\_index | Column\_name | Index\_type |
|-------- |----------- |--------- |-------------- |------------ |----------- |
| question | 0           | PRIMARY   | 1              | id           | BTREE       |
| question | 1           | type      | 1              | type         | BTREE       |

我们来看看在 `type` 上做 `DISTINCT` 和 `GROUP BY` 操作的结果。

```sql
SELECT DISTINCT type FROM question;
```

| type              |
|----------------- |
| choice            |
| single\_choice    |
| uncertain\_choice |
| determine         |
| fill              |
| essay             |
| material          |

```sql
SELECT type FROM questiton GROUP BY type;
```

| type              |
|----------------- |
| choice            |
| single\_choice    |
| uncertain\_choice |
| determine         |
| fill              |
| essay             |
| material          |

完全一样。

MySQL 是怎么实现的呢. Let's `EXPLAIN`.

```sql
EXPLAIN SELECT DISTINCT type FROM question;
```

| id | select\_type | table    | type  | possible\_keys | key  | key\_len | ref  | rows | Extra                    |
|--- |------------ |-------- |----- |-------------- |---- |-------- |---- |---- |------------------------ |
| 1  | SIMPLE       | question | range | type           | type | 1        | NULL | 16   | Using index for group-by |

```sql
EXPLAIN SELECT type FROM question GROUP BY type;
```

| id | select\_type | table    | type  | possible\_keys | key  | key\_len | ref  | rows | Extra                    |
|--- |------------ |-------- |----- |-------------- |---- |-------- |---- |---- |------------------------ |
| 1  | SIMPLE       | question | range | type           | type | 1        | NULL | 16   | Using index for group-by |

有趣的是, `DISTINCT` 查询 is using index for group-by. MySQL 基本上把 `DINSTINCT` 翻译成了 `GROUP BY`.

如果是在 `score` 字段上呢？这次我们先 explain.

```sql
EXPLAIN SELECT DISTINCT score FROM question;
```

| id | select\_type | table    | type | possible\_keys | key  | key\_len | ref  | rows  | Extra           |
|--- |------------ |-------- |---- |-------------- |---- |-------- |---- |----- |--------------- |
| 1  | SIMPLE       | question | ALL  | NULL           | NULL | NULL     | NULL | 37424 | Using temporary |

```sql
EXPLAIN SELECT score FROM question GROUP BY score;
```

| id | select\_type | table    | type | possible\_keys | key  | key\_len | ref  | rows  | Extra                           |
|--- |------------ |-------- |---- |-------------- |---- |-------- |---- |----- |------------------------------- |
| 1  | SIMPLE       | question | ALL  | NULL           | NULL | NULL     | NULL | 37424 | Using temporary; Using filesort |

在没有索引的情况下，MySQL 扫描全表，再作去重或 groupby. 不同的是，`GROUP BY` 使用了 `filesort` 而 `DISTINCT` 没有。

这样使得 `GROUP BY` 的结果是有序的，而 `DISTINCT` 不是. 因为 MySQL 的 `GROUP BY` 默认是有序的。


```sql
SELECT DISTINCT score FROM question;
```

| score |
|----- |
| 1.0   |
| 2.0   |
| 0.0   |
| 3.0   |
| 8.0   |
| 12.0  |
| 6.0   |
| 4.0   |
| 10.0  |
| 14.0  |
| 18.0  |
| 2.5   |
| 15.0  |
| 2.3   |
| 5.0   |
| 1.5   |
| 4.5   |

```sql
SELECT score FROM question GROUP BY score;
```

| score |
|----- |
| 0.0   |
| 1.0   |
| 1.5   |
| 2.0   |
| 2.3   |
| 2.5   |
| 3.0   |
| 4.0   |
| 4.5   |
| 5.0   |
| 6.0   |
| 8.0   |
| 10.0  |
| 12.0  |
| 14.0  |
| 15.0  |
| 18.0  |

MySQL 的执行选择策略还是很复杂的，有没有索引影响的东西其实还是比较多的。

PS: 如果要强制 `GROUP BY` 不排序，使用 `group by # order by null`.
