---
layout: post
title: MySql锁的介绍
categories: [mysql]
description: MyISAM、InnoDB的锁示例解析
keywords: mysql
typora-root-url: ../../
---

## MyISAM中的锁介绍

先来总结，下面用实际案例来验证

 - 共享读锁

 - 独占写锁 

   
### 共享读锁
### 独占写锁

一个session对表student加write锁后，其他session对该表的**读，写请求都会阻塞**。所以叫独占写锁。

看下面的实例：

session1:

```mysql
mysql> lock table student write; -- 步骤1：加write锁
Query OK, 0 rows affected (0.00 sec)

mysql> 
mysql> 
mysql> select * from student; -- 步骤2: 查询
+----+------+
| id | name |
+----+------+
|  1 | a    |
|  2 | b    |
|  3 | c    |
+----+------+
3 rows in set (0.00 sec)

mysql> update student set name='a1' where id=1; 
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from student; 
+----+------+
| id | name |
+----+------+
|  1 | a1   |
|  2 | b    |
|  3 | c    |
+----+------+
3 rows in set (0.00 sec)

mysql> 
mysql> unlock tables; -- 步骤4  解锁，解锁后看下sessino2的步骤5
Query OK, 0 rows affected (0.01 sec)


```

Session2:

```mysql
mysql> select * from student; -- 步骤3，在session2中查询student表会阻塞
+----+------+
| id | name |
+----+------+
|  1 | a1   |
|  2 | b    |
|  3 | c    |
+----+------+
3 rows in set (49 min 55.59 sec) -- 步骤5 不再阻塞，session2返回结果
```

看下关键的几个步骤解释：

- 步骤1: session1对表student**==加write锁==**
- 步骤2. session1对表student做查询
- 步骤3. 开启session2，并在session2中查询student表，此时==**会阻塞**==， 直到student表解锁
- 步骤4.  session解锁student， 
- 步骤5. 此时session2会从阻塞状态返回，继续执行返回查询到结果，可以看到查询用的耗时为：49 min 55.59 sec（ps: 中间去吃了个饭....具体等待多久应该有参数控制）

## InnoDB的锁介绍

### 排他锁

准备表数据，存储引擎engine=InnoDB;

```mysql
CREATE TABLE `student` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(10) DEFAULT NULL,
  KEY `idx_id` (`id`)
) ENGINE=InnoDB;

insert into student values(1,'a'),(2,'b'),(1,'aa'),(3,'c');
```

sessionA:

```mysql
mysql> select @@autocommit;
+--------------+
| @@autocommit |
+--------------+
|            1 |
+--------------+
1 row in set (0.00 sec)

mysql> set autocommit = 0;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@autocommit;
+--------------+
| @@autocommit |
+--------------+
|            0 |
+--------------+
1 row in set (0.00 sec)
mysql> 
mysql> select * from student where id=1 and name='a' for update;
+------+------+
| id   | name |
+------+------+
|    1 | a    |
+------+------+
1 row in set (0.00 sec)

```

SessionB:

```mysql
mysql> set autocommit = 0;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@autocommit;
+--------------+
| @@autocommit |
+--------------+
|            0 |
+--------------+
1 row in set (0.00 sec)
mysql> select * from student where id=1 and name='aa' for update;
+------+------+
| id   | name |
+------+------+
|    1 | aa   |
+------+------+
1 row in set (43.72 sec)


```

InnoDB锁的是索引，如果没有走索引，则退化成锁表。



删除索引：

> drop index idx_id on student;

```mysql
mysql> select * from student where id=1 and name='a' for update; -- 锁表
+------+------+
| id   | name |
+------+------+
|    1 | a    |
+------+------+
1 row in set (0.01 sec)

mysql> commit; -- 10s后commit;
Query OK, 0 rows affected (0.00 sec)
```

```mysql
mysql> select * from student where id=2 and name='b' for update; -- 阻塞
+------+------+
| id   | name |
+------+------+
|    2 | b    |
+------+------+
1 row in set (10.91 sec) -- 阻塞等待了10s
```

### 间隙锁

对某个范围的数据加锁, 其他事物往该范围插入数据时，会阻塞

A: 将 **id >=3 lock in share mode;**大于3的范围加锁

```mysql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from student;
+------+------+
| id   | name |
+------+------+
|    1 | a    |
|    2 | 2b   |
|    1 | aa   |
|    3 | c    |
|    5 | 55   |
|    4 | 44   |
+------+------+
6 rows in set (0.00 sec)

mysql> select * from student where id >=3 lock in share mode;
+------+------+
| id   | name |
+------+------+
|    3 | c    |
|    5 | 55   |
|    4 | 44   |
+------+------+
3 rows in set (0.01 sec)
```



B:插入 ***id=8的记录，会阻塞***

```mysql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from student;
+------+------+
| id   | name |
+------+------+
|    1 | a    |
|    2 | 2b   |
|    1 | aa   |
|    3 | c    |
|    5 | 55   |
|    4 | 44   |
+------+------+
6 rows in set (0.00 sec)

mysql> select * from student;
+------+------+
| id   | name |
+------+------+
|    1 | a    |
|    2 | 2b   |
|    1 | aa   |
|    3 | c    |
|    5 | 55   |
|    4 | 44   |
+------+------+
6 rows in set (0.01 sec)

mysql> insert into student values(8,'888');  # 阻塞
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```





