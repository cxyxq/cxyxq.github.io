---
layout: post
title: MySql事务基本概念
categories: [mysql]
description: MySql事务基本概念
keywords: mysql
typora-root-url: ../../
---

## 事务

事务表示一组原子性的SQL操作，或者说一个独立的工作单元。事务内的语句，要么全部执行成功，要么全部执行失败。

- 原子性 atomicity

  一个事务内的sql操作，要么全部成功，要么全部失败，不可能只支持其中的一部分。

- 一致性 consistency

  数据库总是从一个一致性状态转换到另一个一致性状态。例如转账的经典例子, 如果事务最终没有被提交，事务中做的修改是不会保存到数据库的，一致性可以得到保证。

- 隔离性 isolation

  一个事务中的修改没有被提交前，**"通常来说"**对其他事务是不可见的。因为它与`事务的隔离级别`有关.

- 持久性 durability

  一旦事务成功提交，对其所做的修改就永久保存到了数据库中。



## 隔离级别

先上总结

| 隔离级别         | 脏读可能性 | 不可重复读可能性 | 幻读可能性 | 加锁读 |
| :--------------- | ---------- | ---------------- | ---------- | ------ |
| Read UnCommitted | Yes        | Yes              | Yes        | No     |
| Read Committed   | No         | Yes              | Yes        | No     |
| Repeatable Read  | No         | No               | Yes        | No     |
| Serializable     | No         | No               | No         | Yes    |

1. 脏读：一个事务可以读取到另外一个事务**未提交**的数据。

2. 不可重复读：事务内执行两次相同的查询，可能返回的结果不一样。
3. 幻读：一个事务在读取一个范围内的数据时，另外一个事务往该范围又插入了一条数据， 当之前事务再次进行改返回查询时，会产生幻行(Phantom Row)。可以用MVCC解决幻读的问题。



在Repeatable Read级别下，普通的select确实可以做到可重复读（其他事务改变的数据对当前事务不可见），但是如果***s******elect xxxx lock in share mode***，就会变成不可重复读，会把最新的数据查询出来。



## MVCC

Multiple Version Concurrent Control 多版本并发控制  [MVCC底层讲解-B站视频](https://www.bilibili.com/video/av83518115?from=search&seid=8022441819277038071)

1. 当执行事务的第一条select语句时，就会创建当前session级别的read view(当前库，所有表的read view)。 后续查询都依赖第一次创建的read view来进行版本链查找。 所以Repeatable Read级别支持可重复读
2. 在Read Committed级别，每次都会产生新的read view，所以不可重复读。

注意：begin/start transaction 命令不是一个事务的起点，在执行到第一个操作InnoDB表的语句时，事务才真正启动，才会向mysql申请事务id,mysql内部是严格按照事务的启动顺序来分配事务id的。