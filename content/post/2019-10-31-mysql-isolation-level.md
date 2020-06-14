+++
date = "2019-10-31T00:00:00Z"
tags = ["mysql", "database"]
categories = ["database"]
title = "并发请求与 MySQL 事务隔离"

+++

旧文一则，记录处理数据异常的过程，借此阐述 MySQL 事务隔离的一些相关概念。

## 问题
在线做题是公司一个重要的业务。同一个用户可以对一张试卷(testpaper)有多张答卷(result)，但是最多只能有一张正在进行(doing)的答卷(active result). 最近发现了同一用户一张试卷有两张 active 的 result 的情况。

我们的提交答案接口兼有新开答卷的功能。当发现对于目标试卷当前用户没有答卷记录或者当前答卷状态已经是完成(finished) 状态，会新开一张卷子再提交答案。把这一过程简化一下，可以用以下步骤表示。

1.  find result of which active = 1
2.  unset all active results
3.  create new active result
4.  insert answers

需要注意的是，我们的代码将这个过程放在了一个事务里，虽然是不加锁的。但是 MySQL innodb 会自动给所有 update, insert 操作加上写锁(exclude lock, aka, X lock)。所以除了第一步以外，其它步骤都是有锁的。

什么情况下会造成这个接口的并发呢。一般情况下，请求都是顺序发出的。但是避免不了网络状况不佳等情况，导致两个请求同时到达或者重复调用。

正常情况下是这样的。

| request 1                                      | request 2                                   |
|------------------------------------------------|---------------------------------------------|
| 1. find result of which active = 1(*finished*) |                                             |
| 2. unset all active results                    |                                             |
| 3. create new active result                    |                                             |
| 4. insert answers                              |                                             |
|                                                | 5. find result of which active = 1(*doing*) |
|                                                | 6. insert answers                           |

但是也有可能是这样的。


| request 1                                      | request 2                                      |
|------------------------------------------------|------------------------------------------------|
| 1. find result of which active = 1(*finished*) |                                                |
|                                                | 2. find result of which active = 1(*finished*) |
| 3X. unset all active results                   |                                                |
| 4X. create new active result                   |                                                |
| 5X. insert answers                             |                                                |
|                                                | 6X. unset all active results                   |
|                                                | 7X. create new active result                   |
|                                                | 8X. insert answers                             |

或者这样。

| request 1                                      | request 2                                      |
|------------------------------------------------|------------------------------------------------|
| 1. find result of which active = 1(*finished*) |                                                |
|                                                | 2. find result of which active = 1(*finished*) |
|                                                | 3X. unset all active results                   |
|                                                | 4X. create new active result                   |
|                                                | 5X. insert answers                             |
| 6X. unset all active results                   |                                                |
| 7X. create new active result                   |                                                |
| 8X. insert answers                             |                                                |

因为 unset all active results 这一步需上一个写锁，仅一个事务能拿到，另一个必须等待，而且事务中的锁在提交时才会释放。所以仅可能出现以上三种情况。

在第一步 find active result 之后人为加入延迟数秒，以便手动触发的先后请求能够模拟并发操作，得到以下结果。

```sql
SELECT id, testId, userId, active, status FROM testpaper_result ORDER BY id DESC limit 5;
```

|      id | testId | userId | active | status    |
|---------|--------|--------|--------|-----------|
| 1227562 |   1594 | 238823 |      1 | doing     |
| 1227561 |   1594 | 238823 |      0 | doing     |
| 1227560 |   1594 | 238823 |      0 | finished  |
| 1227559 |   1594 | 238823 |      0 | finished  |
| 1227558 |   1145 | 120918 |      1 | reviewing |

可以看到，多了一行 active = 0 但是 status = doing 的记录。这是符合并发预期的，因为两个请求都认为自己应该将其它所有的 result unset 掉，然后新建一张答卷，所以第二个请求将第一个请求的结果 unset 掉了，又新建了一张。这造成了一条作废的记录，不符合业务预期，也可能导致第一次请求携带的答案丢失。


## 事务隔离

我们已经在代码中使用了事务了，难道事务不应该就是支持 ACID，即原子性，一致性，隔离性，持久性吗？在这里，主要的关注点是隔离性。理想的情况下两个并发的事务执行的结果应该等价于两个事务串行化。然而，实际的数据库管理系统却没有做到，或者没有默认做到，即便 SQL 标准要求是这样的。

举 MySQL 为例，必须将事务隔离等级手动设置成可串行化，innodb 会自动给每一个 SELECT 语句加上读锁(share lock, aka, S lock). 这时候，两个请求变成这样。

| request 1                                       | request 2                                       |
|-------------------------------------------------|-------------------------------------------------|
| 1S. find result of which active = 1(*finished*) |                                                 |
|                                                 | 2S. find result of which active = 1(*finished*) |
|                                                 | 3X. unset all active results(*DEAD LOCK*)       |
|                                                 | *ROLL BACK*                                     |
| 6X. unset all active results                    |                                                 |


可以看到因为读锁之间互相不互斥，一直到 2S 都是没有问题的。但是 3X 需要获取写锁，与 1S 互斥，所以 3X 必须等待请求 1 执行完才能继续。然而 6X 必须等待请求 2 执行完。这就造成死锁。MySQL 检测到死锁后会回滚请求 2, 让请求 1 先执行完。可以看到，这种方式虽然达到了目的，但是死锁检测，回滚事务，一些非必要的加锁，都浪费了大量时间和资源，性能问题导致这种理论上完美的解决方案没有推行。

MySQL 还支持另外三个事务隔离等级，分别是未提交读，提交读，可重复读，我们分别来看一下。

未提交读会读取其它事务未提交的结果，总是读取最新结果，一般来说是不可靠的，又名浏览性读。

提交读会读取事务开始时的数据快照，也即事务开始时其它事务已经提交的结果。未提交读和提交读默认都是非锁定的，是通过数据库的数据版本管理实现的。从前面的例子可以看到，仅仅是这样还是不够的。必须有锁的辅助。除了 INSERT, UPDATE, DELETE 语句会自动加上写锁，对于 SELECT 语句，可以用 SELECT ... LOCK IN SHARE MODE, SELECT ... FOR UPDATE 的方式加上读锁或者写锁。在提交读隔离等级下，innodb 会将选中的行锁定。我们看看是怎么通过写锁来解决之前的问题。

| request 1                                       | request 2                                    |
|-------------------------------------------------|----------------------------------------------|
| 1X. find result of which active = 1(*finished*) |                                              |
| 2X. unset all active results                    |                                              |
| 3X. create new active result                    |                                              |
| 4X. insert answers                              |                                              |
|                                                 | 5X. find result of which active = 1(*doing*) |
|                                                 | 6X. unset all active results                 |
|                                                 | 7X. create new active result                 |
|                                                 | 8X. insert answers                           |

因为我们选中的 active result 有可能是要更新的，所以我们预先加了一个写锁，于是请求 2 的 5X 被阻塞，2X 也就不用等 request 2 结束，不会死锁，顺利结束。

到此为止，问题得到了解决。什么时候应该加写锁，什么时候应该加读锁呢？如果在同一事务的后续操作中，记录有可能要修改，那就应该加上写锁。如果希望事务结束时，记录仍然保持不变，那就应该加上读锁。

那么可重复读用来克服什么问题呢？我们想象一下有一个未知的操作，比如来自另一个客户端的请求，插入了一条新的记录。

| APP request                                     | website request              |
|-------------------------------------------------|------------------------------|
| 1X. find result of which active = 1(*finished*) |                              |
| 2X. unset all active results                    |                              |
|                                                 | 3X. create new active result |
| 4X. create new active result                    |                              |
| 5X. insert answers                              |                              |


那么会得到这样的结果。

|      id | testId | userId | active | status    |
|---------|--------|--------|--------|-----------|
| 1227562 |   1594 | 238823 |      1 | doing     |
| 1227561 |   1594 | 238823 |      1 | doing     |
| 1227560 |   1594 | 238823 |      0 | finished  |
| 1227559 |   1594 | 238823 |      0 | finished  |
| 1227558 |   1145 | 120918 |      1 | reviewing |

出现了两条 active 的 doing result. 为什么会这样呢？因为提交读的锁锁定的是具体的行，并不阻止新的记录产生。APP request 如果在 4X 前面重复了 1X 的查询，会发现前后读到的结果是不一样的，这就是幻读。为了达到避免这种情况，必须做到可重复读。innodb 使用行锁加区间锁来实现。这对事务的正确实现是非常重要的，而且相对提交读也没有太大性能负担，具体可见《MySQL 技术内幕》的论述。但值得注意的是，可重复读的区间锁分析起来相对复杂，对开发人员提出了更多的要求。


## 接口设计

并发请求除了在数据库处理好事务以外，在客户端和服务器的交互上也需要注意。

首先要注意的是 restful 的接口设计。restful 常用的动作是 GET, POST, PUT, PATCH, DELETE. 其中 GET, PUT, DELETE 都应当是幂等的，也即多次请求对数据的影响与一次请求一致。POST 与 PATCH 则不符合。对于这两个接口，都要特别注意重复调用接口造成的影响。有时候需要客户端收到服务器的回复后再继续后面的请求。

在我们之前的例子中，由于新加答案的请求中混杂了新开答卷的操作，所以给业务逻辑也造成了困难。语义明确，用途单一的接口也有利于并发处理。


## 并发测试与调试

并发是测试与调试的难点。模拟并发对于代码触发和手工测试都是困难的事情。出了问题一般只能找线上日志。一个能够保存事发现场而且方便事后查找的日志是起码的支撑。
