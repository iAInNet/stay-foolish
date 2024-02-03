---
layout: post
title: "对 MySQL InnoDB 可重复读隔离级别是否解决幻读问题的定性"
date: 2024-02-03 00:00:00 +000
categories: 分布式存储系统设计与实现
---

本篇文章会从 MySQL InnoDB 实现的可重复隔离级别所具备的特定来回答，“MySQL InnoDB 可重复读隔离级别是否解决幻读问题？”。更重要地是，理解“为什么 MySQL InnoDB 的可重复读隔离级别会是如此？”。

关键词：MySQL、可重复读隔离级别、幻读问题

## MySQL InnoDB 可重复读隔离级别和幻读问题

先看 MySQL InnoDB 可重复读隔离级别具备的特性：

“快照读”。那么在第一次 select 执行的时候就会拿到读视图（read-view），同一个事务看到的数据是一致的，期间有别的事务进行更新、插入操作也同样是不可见。可以避免“P3 幻读问题”。

“当前读”。也就是执行 select for update/share。这是在读取的时候就上锁，也会用到 next-key lock 确保该事务执行期间哪怕有别的事务执行插入操作也同样会被阻塞，换句话说，当前读通过上锁操作阻塞其他事务的写操作，达到了“串行化隔离级别（Serializable Isolation）”的效果。可以避免“P3 幻读问题”。

“快照读”和“当前读”混用。当执行“当前读”的时候，MySQL InnoDB 会返回最新的已提交的结果。这就导致期间如果发生了别的事务的插入操作，那么就会导致这一事务前后两次读取拿到不一致结果。这种场景下，读取操作等同于“读已提交隔离级别”的效果，哪怕只是期间提交更新操作，也会拿到不一致的结果，所以甚至破坏了 ANSI SQL 对于可重复读隔离级别的要求。简单来说，不仅没有避免“P3 幻读问题”，甚至于连“P2 不可重复读问题”都出现了。

“读写混合场景”。更新操作，要给最新已提交的结果上锁，并且 update 执行的结果，肯定对事务内可见。这就导致一旦更新成功，本事务内的 read-view 就会被标记失效，那么同一个事务下一次查询就会看到新的数据。当然这种不属于标准定义中“P3 幻读问题”所描述的场景，暂且不用考虑。

所以，按照作者本人的理解。从学术标准定义出发，那么“快照读”和“当前读”混用场景，MySQL InnoDB 的可重复读隔离级别，甚至连“P2 不可重复读问题”都无法避免，就更不用谈“P3 幻读问题”。所以，这个问题如果只是讨论到“是否解决幻读问题”，显然是不够的。还需要接着往下深究，到底 MySQL InnoDB 的可重复读隔离级别是什么样的？

当然，对于是否解决幻读问题的不同观点感兴趣的可以移步到这里：[InnoDB 中 RR 隔离级别能否防止幻读？ - github issuse](https://github.com/Yhzhtk/note/issues/42)。

接下来，我将用示例，在 MySQL 和 PostgreSQL 两个系统上演示“快照读”和“当前读”混用场景。通过对比，发现 MySQL InnoDB 的可重复读隔离级别，到底做了什么以及为什么会这样设计。

## MySQL InnoDB 可重复读隔离级别，在快照读和当前读混用场景，连可重复读都无法保证

“快照读”和“当前读”混合。因为快照读执行的时候不会加锁，那么就不会阻塞其他事务操作，假设这期间有其他事务执行并提交了，此时本事务再执行当前读，就会拿到最新已提交的结果，这就相当于在一个事务里读到别的事务已提交的结果。此时，别说避免“P3 幻读问题”，连“P2 不可重复读问题”都出现了。

测试用例表的数据初始状态：

```text
+----+------+
| id | val  |
+----+------+
|  1 |   11 |
|  2 |   22 |
+----+------+
```

先看 “P3 幻读问题”：

| T1 | T2 |
| ---- | ---- |
| SET SESSION AUTOCOMMIT = 0; | SET SESSION AUTOCOMMIT = 0; |
| begin; | begin; |
| select id, val from txn_demo where id >= 2; |  |
| -- reuslt: (2, 22) |  |
|  | insert into txn_demo(id, val) values(3, 33); |
|  | commit; |
| select id, val from txn_demo where id >= 2 for update; |  |
| -- result: (2, 22), (3, 33)；并未得到一致性的结果 |  |
| select id, val from txn_demo where id >= 2; |  |
| -- reuslt: (2, 22) |  |

可以看出，MySQL InnoDB 执行“当前读”的时候，会读取到别的事务已经插入的新数据，并未得到一致性的结果，没有避免“P3 幻读问题”。

那么，再扩展下，将 `insert` 操作改成 `update`，再看以下示例：

| T1 | T2 |
| ---- | ---- |
| SET SESSION AUTOCOMMIT = 0; | SET SESSION AUTOCOMMIT = 0; |
| begin; | begin; |
| select * from txn_demo where id = 1; |  |
| -- reuslt: 11 |  |
|  | update txn_demo set val = 12 where id = 1; |
|  | commit; |
| select * from txn_demo where id = 1 for update; |  |
| -- result: 12；并未得到一致性的结果 |  |
| select * from txn_demo where id = 1; |  |
| -- result: 11 |  |

可以看出，MySQL InnoDB 执行“当前读”的时候，哪怕另外一个事务执行的数据更新操作（事务 T2 是 `update` 操作），在这一个事务里两次读取，拿到的结果也会是不一致的，也就是说 MySQL InnoDB 可重复读隔离级别，甚至无法避免“P2 不可重复读”问题。

那么，可以下结论：“**MySQL InnoDB 可重复读隔离级别，名不副实，连可重复读都无法做到吗？**”

在回答这个问题之前，可以先看下 PostgreSQL 遇到这种场景又是如何处理的？

## PostgreSQL 可重复读隔离级别，在快照读和当前读混用场景，不会有幻读问题也不会有可重复读问题

与 MySQL InnoDB 实现对比，同样的场景，PostgreSQL 就不会出现“P3 幻读问题”也不会有“P2 不可重复读问题”。

测试用例表的数据初始状态：

```text
+----+------+
| id | val  |
+----+------+
|  1 |   11 |
|  2 |   22 |
+----+------+
```

同样地，先看 “P3 幻读问题”：

| T1 | T2 |
| ---- | ---- |
| begin; |  |
| set default_transaction_isolation='repeatable read'; | begin; |
|  | set default_transaction_isolation='repeatable read'; |
| select * from txn_demo where id >= 2 |  |
| -- result: (2, 22) |  |
|  | insert into txn_demo(id, val) values(3, 33); |
|  | commit; |
| select * from txn_demo where id >= 2 for update; |  |
| -- result: (2, 22)；获取到一致性结果，并未看到事务 T2 新写入的数据 |  |
| select * from txn_demo where id >= 2; |  |
| -- result: (2, 22) |  |
| commit; |  |

可以看出，PostgreSQL 不会因为 `select...for update` 要上写锁，就返回最新插入的结果 `id=3`，从读取来看依然保持了一致性的结果，两次都是只有一条记录 `(2, 22)`。可以避免“P3 幻读问题”。

当然，需要注意的是，此时事务 T1 虽然看不到 (3, 33) 这条记录，但是也已经成功给该记录上锁了，如果另外一个事务 T3 要想修改这条记录，就会被阻塞。

再将 `insert` 换成 `update` 操作，看看是否会有“不可重复读问题”：

- 同样地，将两个事务 T1 和 T2 的隔离级别设定为“可重复读（repeatable read）”
- 事务 T1 快照读，拿到结果 `(1, 11)`
- 事务 T2 提交更新操作将 `id=1` 更新为 12
- 事务 T1 快照读，拿到结果依然是 `(1, 11)`
- 事务 T1 执行当前读（select...for update），此时数据库系统会报错：
	- `ERROR:  could not serialize access due to concurrent update`
	- 事务 T2 修改了 `id=1` 的记录并且已提交，存在并发写入的情况，事务 T1 如果拿着快照数据（老版本）做修改再提交，就有出现“丢失更新问题（Lost Updates）”的风险。所以当事务 T1 要上写锁的时候，被阻塞回滚了

| T1 | T2 |
| ---- | ---- |
| begin; |  |
| set default_transaction_isolation='repeatable read'; | begin; |
|  | set default_transaction_isolation='repeatable read'; |
| select id, val from txn_demo where id = 1; |  |
| -- reuslt: (1, 11) |  |
|  | update txn_demo set val = 12 where id = 1; |
|  | commit; |
| select id, val from txn_demo where id = 1; |  |
| -- reuslt: (1, 11)；快照读拿到一致性结果 |  |
|  |  |
| select id, val from txn_demo where id = 1 for update; |  |
| -- ERROR:  could not serialize access due to concurrent update |  |
|  |  |
| select * from txn_demo where id = 1; |  |
| -- ERROR:  current transaction is aborted, commands ignored until end of transaction block |  |
| commit; |  |
| -- ROLLBACK |  |

不论如何，事务 T1 整个事务生命周期并未出现不一致读的情况。也就是说，同样避免“P2 不可重读取问题”。当然，代价就是事务 T1 需要回滚重新执行。

## MySQL InnoDB 可重复读隔离级别的实现，体现了工程实现和学术标准定义之间的错位

回到一开始的结论：“**MySQL InnoDB 可重复读隔离级别，名不副实，连可重复读都无法做到吗？**”

我觉得，这显然也是不妥当的。

要从工程实现上去梳理 MySQL InnoDB 在可重复读隔离级别，“快照读”和“当前读”混用的情况，出现“P2 不可重复读”和“P3 幻读问题”问题的原因。

首先需要明白一点，在做 select...for update 的时候就要上锁：表级别意图写锁（IX）和行级别写锁（X），跟 update 操作上的锁是一样的。从上锁的角度来看 select...for update 相比于 update 操作，两者并无区别。

在“快照读”和“当前读”混用场景，发生“当前读”的时候，数据库系统有三种选择：
- 第一种，不管其他事务写入情况继续返回快照结果，并让事务继续执行。这样就确保了“可重复读”的特性，也不会出现“幻读”。但新的问题出现了，这个事务已经上了写锁但是事务内又是基于过期数据进行操作，那就有“丢失更新问题”风险。
- 第二种，直接报错。不允许获取该资源的写锁，直接回滚事务。这是 PostgreSQL 的做法。
- 第三种，阻塞等待并拿到最新写入结果。事务后续对最新的结果再做写操作，也不会有“丢失更新问题”，当然带来的问题就是这一个事务生命周期内读取结果不一致。这是 MySQL InnoDB 采用的做法。

所以，“快照读”和“当前读”混用所带来的问题。归其本质，是 MySQL InnoDB 对于“丢失更新问题”处理上，在工程实现和学术标准之间所做出的取舍。它不准备回滚事务来避免“丢失更新问题”，而是返回最新结果，这会带来不一致的读取结果，但是也让整体性能有所提升，毕竟回滚事务是有很大系统开销。

作者个人会更喜欢 PostgreSQL 的做法，它让隔离级别的定义以及所能解决的问题边界更加清晰，很干净，不会像 MySQL 那样更加严格的隔离级别还出现了低层级的隔离错误。但不能完全否定 MySQL 这么做的理由，毕竟从使用场景出发，都用 `select...for update` 上写锁了，那么拿到最新结果进行后续更新操作反而是代价更小的实现方式，绝大多数情况也不会关注上一次读取结果跟这一次不一样，使用最新结果继续操作就好了。

因此，MySQL InnoDB 的可重复读隔离级别，没办法按照 ANSI SQL 定义的可重复隔离级别（Repeatable Read）来衡量，也不完全符合后来改进的快照隔离级别（Snapshot Isolation）的定义（A Critique of ANSI SQL Isolation Levels，Berenson）。从特性角度来看待，它在两者之间。

## 总结

MySQL InnoDB 的可重复隔离级别，具备避免“P3 幻读问题”的功能特性：（1）只用快照读；（2）先用“当前读”。但系统本身并无意愿要在这个隔离级别解决幻读问题，导致应用层可以通过不同的使用方式避免或者复现幻读问题。

从学术标准定义出发，并没有解决“P3 幻读问题”，甚至于它所实现的“可重复读隔离级别”，和 ANSI SQL 标准定义的“可重复读隔离级别”有所不同，也跟快照隔离级别（Snapshot Isolation）不一样。只能说，这是 MySQL 实现的“可重复读隔离级别”，并具备这些特性。

所以，在生产使用过程中注意甄别，避免误用带来非预期的结果：
- 不要混用快照读和当前读：
	- 快照读，仅用于只读事务，可避免“P3 幻读问题”
	- 当前读，用于有写操作事务，可避免 “P3 幻读问题”
- 混用快照读和当前读的情况，也要先做”当前读“，这样可以确保整个事务启动就上写锁，也就能避免“P3 幻读问题”

## 参考文献

- [How to produce "phantom read" in REPEATABLE READ? (MySQL)](https://stackoverflow.com/questions/5444915/how-to-produce-phantom-read-in-repeatable-read-mysql)
- [InnoDB 中 RR 隔离级别能否防止幻读？ - github issuse](https://github.com/Yhzhtk/note/issues/42)
- [MySQL 可重复读隔离级别，解决幻读了吗？](https://www.xiaolincoding.com/mysql/transaction/phantom.html)
- [MySQL-当前读、快照读、MVCC](https://www.jianshu.com/p/eb3f56565b42)
- [Testing transaction isolation levels - github](https://github.com/ept/hermitage)
- [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf)
