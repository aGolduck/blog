+++
date = "2020-05-17T00:00:00Z"
tags = ["database", "ORM"]
categories = ["database", "javascript"]
title = "typeorm 数据库自动迁移"

+++

在小型应用以及应用原型快速开发阶段，关系数据库表定义
自动迁移是非常方便的特性。现在成熟的 ORM 都有所支持。
以 typorm 为例，一般来说，我们进行一次成功的数据库迁移需要包括下面几步。

1. 使用 `typeorm migration:generate -n` 自动比较代码与数据库的表头定义，生成迁移脚本。
2. 编辑迁移脚本，使迁移过程更合理，更数据安全。迁移过程往往不只是对表定义的修改，
还涉及到数据的处理，有必要时可用 `typeorm migration:create` 生成空脚本编辑
3. `typeorm migration:run` 运行迁移脚本。typeorm 会自动创建一个迁移过程元数据表，
   记录已执行的脚本，以此在后续过程判断应该从执行哪些迁移脚本。
4. 有必要的情况下可用 `typeorm migration:revert` 回滚。

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
但是对于生产环境的布署，最重要的还有布署失败自动回滚的问题，
这和数据库的事务是类似的。一般我们使用的数据库事务指的是
DML(数据操作语言，如数据增删改查）的事务。那么对于 DML(数据定义语言)
能不能做到相同的原子性呢，请听
[下回](https://wpchou.github.io/post/2020-05-18-atomic-ddl)
分解。
