---
title: MySQL锁机制innoDB
created: '2023-09-29T09:37:08.725Z'
modified: '2023-09-29T12:28:51.090Z'
---

# MySQL锁机制innoDB

InnoDB通过给索引上的索引项加锁来实现行锁。所以，只有通过索引条件检索数据，InnoDB 才使用行锁，否则，InnoDB 将使用表锁。

其他注意事项：
- 在不通过索引条件查询的时候，InnoDB使用的是表锁，而不是行锁。
- 由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以即使是访问不同行的记录，如果使用了相同的索引键，也是会出现锁冲突的。
- 当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。
- 即便在条件中使用了索引字段，但具体是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，需要检查SQL的执行计划，以确认是否真正使用了索引。

InnoDB 支持多粒度锁（multiple granularity locking），它允许行级锁与表级锁共存，而意向锁就是其中的一种表锁。

#### 意向锁（Intention Locks）
意向锁是一种不与行级锁冲突的表级锁。意向锁分为两种：

- 意向共享锁 （intention shared lock, IS）：事务有意向对表中的某些行加 共享锁 （S锁） -- 事务要获取某些行的 S 锁，必须先获得表的 IS 锁。 SELECT column FROM table ... LOCK IN SHARE MODE;

- 意向排他锁 （intention exclusive lock, IX）：事务有意向对表中的某些行加 排他锁 （X锁） -- 事务要获取某些行的 X 锁，必须先获得表的 IX 锁。 SELECT column FROM table ... FOR UPDATE;

意向锁是有数据引擎自己维护的，用户无法手动操作意向锁，在为数据行加共享 / 排他锁之前，InooDB 会先获取该数据行所在在数据表的对应意向锁。

如果另一个任务试图在该表级别上应用共享或排它锁，则受到由第一个任务控制的表级别意向锁的阻塞。第二个任务在锁定该表前不必检查各个页或行锁，而只需检查表上的意向锁。


#### 隐式加锁
- InnoDB自动加意向锁。对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据表加意向排他锁；
- 对于普通SELECT语句，InnoDB不会加任何锁；

#### 显示加锁：
- 共享锁（S）：`SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE`
- 排他锁（X) ：`SELECT * FROM table_name WHERE ... FOR UPDATE`

用 `SELECT … IN SHARE MODE` 获得共享锁，主要用在需要数据依存关系时来确认某行记录是否存在，并确保没有其他对这个记录进行 UPDATE 或者 DELETE 操作。

但是如果当前事务也需要对该记录进行更新操作，则很有可能造成死锁，对于锁定行记录后需要进行更新操作的应用，应该使用 `SELECT… FOR UPDATE` 方式获得排他锁。

InnoDB 如何加表锁：
在用 `LOCK TABLES` 对 InnoDB 表加锁时要注意，要将 `AUTOCOMMIT` 设为 0，否则 MySQL 不会给表加锁；事务结束前，不要用 `UNLOCK TABLES` 释放表锁，因为 `UNLOCK TABLES` 会隐含地提交事务；`COMMIT` 或 `ROLLBACK` 并不能释放用 `LOCK TABLES` 加的表级锁，必须用 `UNLOCK TABLES` 释放表锁。

