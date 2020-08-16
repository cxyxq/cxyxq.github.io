---
layout: post
title: mysql日常总结
categories: [mysql]
description: 记录mysql的日常总结
keywords: mysql
typora-root-url: ../../
---

### count(1)、count(col)、count(*)的区别

先来结论：

- count(*) 统计所有结果集的**所有行数**，包含全null的行
- count(col) 统计**col列不为空**(non-null)的行数
- count(1)与count(*)无区别，都是**统计所有的行数**，因为**1是一个非空的表达式**

#### 解释：count(expr)的作用：

1. 统计某个列(值为非空)的数量
   统计时要求该列的值不能为NULL值
   
2. 统计expr表达式的值不等于null的数量

   count(1)，指的是expr的值等于1，而1不是null，所以会统计所有的行数

3. 统计结果集的行数

   当我们count(*)时，mysql并不会将通配符\*扩展为所有的列，它会忽略所有的列而**直接统计所有行数**。

#### 实验

创建表:

```mysql
-- id,name,age都可以为null
CREATE TABLE `count_test` (
  `id` int(11) DEFAULT NULL COMMENT 'id',
  `name` varchar(16) DEFAULT NULL COMMENT 'name',
  `age` tinyint(3) unsigned DEFAULT NULL COMMENT 'age'
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

表数据：
![](/images/mysql/count_test_rows.png)

执行下面的count查询:

```mysql
-- 统计总行数
select count(*) from count_test; -- 5
select count(1) from count_test;  -- 5

-- 统计列值
select count(id) from count_test; -- 2
select count(name) from count_test; -- 0

-- 统计表达式的值
select count(id > 0) from count_test; -- 2
select count(666) from count_test; -- 5

-- mysql select时检索的每一行数据,对该表达的值都是null
select count(null) from count_test; -- 0
```

参考:

[https://stackoverflow.com/questions/3003457/count-vs-countcolumn-name-which-is-more-correct](https://stackoverflow.com/questions/3003457/count-vs-countcolumn-name-which-is-more-correct)

[MySQL统计数据count(*) 和 count（1） 什么区别？](https://segmentfault.com/q/1010000000761427)

