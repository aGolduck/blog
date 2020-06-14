+++
date = "2019-11-07T00:00:00Z"
tags = ["sequelize", "mysql", "nodejs"]
categories = ["database"]
title = "如何使用 Sequelize (自动)管理数据库事务"

+++

到目前为止，`Sequelize` 仍然是 nodejs 最成熟的 ORM 库。相关的文档有 manual 和 api rereference. 但老实说，写得不算好，manual 只能算 tutorial. api reference 也很多语焉不详，很多用法甚至要去看源码。Sequelize 是很早便成型的库，以 Promise 为核心设计，目前代码没有使用 async function, 文档也没有相关说明。 这倒也不影响我们使用 async function, 但是因为缺乏相关的文档，有时需要自己连蒙带猜的，尤其是事务的使用。本文介绍一下如何使用 Sequelize 的事务 (transaction), 重点讲一下如何实现事务的自动管理。其中，保存点(savepoint)相关的内容在中文互联网未能搜索到相关资料。

## SQL 语句

讨论 Sequelize 之前，我们先来看一下 SQL 和事务相关的语句。不同数据库可能有所不同，下面以 MySQL 为例。
```sql
START TRANSACTION;   // 显式地开启一个事务
SET TRANSACTION READ UNCOMMITTED|READ COMMITTTED|REPEATABLE READ|SERIALIZABLE; // 设置当前事物的隔离级别
COMMIT; // 提交事务
ROLLBACK;  // 回滚事务
SAVEPOINT sp1; // 事务中间提交创建保存点 sp1，可以创建多个保存点
ROLLBACK TO sp1; // 回滚至保存点 sp1
REREASE SAVEPOINT sp1;  // 删除保存点 sp1
```
MySQL 使用以上语句管理事务的生命周期，除了删除保存点， Sequelize 支持所有的语句。

## 手动管理事务
文档给出的例子如下，我在其间加了四个函数调用表示其它的同步语句。

```javascript
return sequelize.transaction().then(t => {
  // f1()
  return User.create({
    firstName: 'Bart',
    lastName: 'Simpson'
  }, {transaction: t}).then(user => {
    // f2()
    return user.addSibling({
      firstName: 'Lisa',
      lastName: 'Simpson'
    }, {transaction: t});
  }).then(() => {
    // f3()
    return t.commit();
  }).catch((err) => {
    // f4()
    return t.rollback();
  });
});
```

这种写法其实很容易让人回忆起 callback hell. 如果换成 async/await 写法就漂亮得多。

### async/await

```javascript
const t = await sequelize.transaction()
try {
  f1()
  const user = await User.create({
    firstName: 'Bart',
    lastName: 'Simpson'
  }, {
    transaction: t
  })
  f2()
  await user.addSibling({
    firstName: 'Lisa',
    lastName: 'Simpson'
  }, {
    transaction: t
  })
  f3()
  await t.commit()
} catch (error) {
  f4()
  await t.rollback()
}
```

这就是最基本的事务处理过程了。

### 多事务

多事务并行也非常简单。

```javascript
const t1 = await sequelize.transaction()
const t2 = await sequelize.transaction()
try {
  await Promise.all([
  User.create({
    firstName: 'Bart',
    lastName: 'Simpson'
  }, {
    transaction: t1
  }),
  User.create({
    firstName: 'Lisa',
    lastName: 'Simpson'
  }, {
    transaction: t2
  })
  ])
  await t1.commit()
  await t2.commit()
} catch (error) {
  await t1.rollback()
  await t2.rollback()
}
```

但实际应用中，很少会出现在一个调用堆栈里出现多事务的，更多的是并发请求造成的，要尤其小心锁的运用。

### 保存点
保存点没有相关的文档说明，只能是看源码和相关的测试文件。首先看 [Transaction 类的构造函数](https://github.com/sequelize/sequelize/blob/bd59b8729b8d898d2f3ce8deb764777bab7e57d5/lib/transaction.js#L37).

```javascript
class Transaction {
  constructor(sequelize, options) {
    // ...
    this.savepoints = []
    // ...
    this.parent = this.options.transaction;

    if (this.parent) {
      this.id = this.parent.id;
      this.parent.savepoints.push(this);
      this.name = `${this.id}-sp-${this.parent.savepoints.length}`;
    } else {
      this.id = this.name = generateTransactionId();
    }
    // ...
  }
}
```
很明显，这段代码的意思是当构造参数 options 有 transaction 属性的时候，不开启新的事务(generateTransactionId)，而仅是向其“父事务”的保存点队列推入一个新的保存点。可以推测，保存点的创建方法如下。

```javascript
const t = await sequelize.transaction()
try {
  const user = await User.create({
    firstName: 'Bart',
    lastName: 'Simpson'
  }, {
    transaction: t
  })
  const savepoint = await sequelize.transaction({transaction: t})
  await user.addSibling({
    firstName: 'Lisa',
    lastName: 'Simpson'
  }, {
    transaction: savepoint
  })
  await t.commit()
} catch (error) {
  await t.rollback()
}
```

可以在[测试文件](https://github.com/sequelize/sequelize/blob/7a90df5cd009024493a306f9a1314d9cbfcb1176/test/integration/sequelize.test.js#L1425)里得到验证。尽管测试代码使用的是回调方式，但和上面的 async/await 示例没有本质区别。

```javascript
describe('transaction', () => {
  // ...
  it('supports nested transactions using savepoints', function() {
    const User = this.sequelizeWithTransaction.define('Users', { username: DataTypes.STRING });

    return User.sync({ force: true }).then(() => {
      return this.sequelizeWithTransaction.transaction().then(t1 => {
        return User.create({ username: 'foo' }, { transaction: t1 }).then(user => {
          return this.sequelizeWithTransaction.transaction({ transaction: t1 }).then(t2 => {
            return user.update({ username: 'bar' }, { transaction: t2 }).then(() => {
              return t2.commit().then(() => {
                return user.reload({ transaction: t1 }).then(newUser => {
                  expect(newUser.username).to.equal('bar');
                  return t1.commit();
                });
              });
            });
          });
        });
      });
    });
  });
  // ...
})
```

测试文件还展示了如何回滚至保存点，实际和前面的事务(`await savepoint.rollback()`)回滚一致。代码过长，下面仅摘录一部分，可以自行到 [github](https://github.com/sequelize/sequelize/blob/7a90df5cd009024493a306f9a1314d9cbfcb1176/test/integration/sequelize.test.js#L1425) 查看。

```javascript
describe('transaction', () => {
  // ...
  it('supports nested transactions using savepoints', function() { // ... });
  it('rolls back to the first savepoint, undoing everything', function() { // ... });
  it('rolls back to the first savepoint, undoing everything', function() { // ... });
  it('supports rolling back a nested transaction', function() { // ... });
  it('supports rolling back outermost transaction', function() { // ... });
  // ...
})
```

### 隔离级别

最后，在获取事务对象的时候可以设置隔离级别，但并不建议这么做。隔离级别最好是由 DBA 统一设置。至此，就介绍完了所有的事务相关的管理语句。

```javascript
const transaction = await sequelize.transaction({
  isolationLevel: Sequelize.Transaction.ISOLATION_LEVELS.SERIALIZABLE
})
```

## 自动管理事务
### 自动提交/回滚

由前面的例子看，所有的代码都要做 commit 和 rollback 的处理，Sequelize 能不能把这件事也做了呢。文档给出的例子如下。

```javascript
return sequelize.transaction(t => {
  return User.create({
    firstName: 'Abraham',
    lastName: 'Lincoln'
  }, {transaction: t}).then(user => {
    if () {
      // Woops, the query was successful but we still want to roll back!
      throw new Error();
    }
  });
}).then(result => {
  // ...
}).catch(error => {
  // ...
});
```

我们转成 async/await 方式。

```javascript
try {
  const result = await sequelize.transaction(async t => {
    const user = User.create({
      firstName: 'Abraham',
      lastName: 'Lincoln'
    }, {
      transaction: t
    })
    if () {
      // Woops, the query was successful but we still want to roll back!
      throw new Error();
    }
    return user
  })
} catch (error) {
  // handle error
}
```

也就是说我们只要把一个 async function 作为 sequelize.transaction 方法的回调函数就可以自动提交/回滚。注意这种写法不能再手动提交/回滚。如果要强制回滚，抛出一个错误即可。

##  自动将查询加入事务
前面的例子，虽然已经实现在自动提交/回滚，但是在每个查询中仍然需要传入 `{transaction: t}`. Sequelize 使用了 CLS 来实现自动插入事务。

文档例子如下。

```javascript
const cls = require('cls-hooked');
const namespace = cls.createNamespace('my-very-own-namespace');
const Sequelize = require('sequelize');
Sequelize.useCLS(namespace);

const sequelize = new Sequelize(....);

await sequelize.transaction(async () => {
  await User.create({name: 'Alice'})
})
```

### 复杂场景：并行/部分事务

下面的例子来自文档，改用 async/await 写法。

```javascript
await sequelize.transaction(async (t1) => {
  await sequelize.transaction(async (t2) => {
    // With CLS enable, queries here will by default use t2
    // Pass in the `transaction` option to define/alter the transaction they belong to.
    await  Promise.all([
        User.create({ name: 'Bob' }, { transaction: null }), // 不使用事务
        User.create({ name: 'Mallory' }, { transaction: t1 }), // 在事务 t1 中执行
        User.create({ name: 'John' }) // this would default to t2
    ]);
  });
});
```

多事务并行其实使用手动管理更好。

### 复杂场景：保存点
sequelize 可传 options 和回调函数两个参数，都是可选的。用同时传两个参数的方法实现保存点。
```javascript
await sequelize.transaction(async (transaction) => {
  await sequelize.transaction({transaction}, async(savepoint) => {
    // ...
  })
})
```
