---
layout: post
title: Spark理论知识总结
categories: [Spark]
description: Spark的运行原理，集群角色等
keywords: Spark
typora-root-url: ../../
---

## Spark

![](/images/spark/spark-cluster-arch.png)

### 集群架构

关于这个架构有几个有用的东西需要注意:

- 每个应用程序都有自己的executor进程，这些进程在整个应用程序运行期间保持运行，并在多个线程中运行任务。这样做的好处是在调度端(每个驱动程序调度自己的任务)和执行器端(来自不同jvm中运行的不同应用程序的任务)隔离应用程序。但是，这也意味着如果不将数据写入外部存储系统，就不能在不同的Spark应用程序(SparkContext的实例)之间共享数据。

- Spark与底层集群管理器无关。只要它能够获得执行程序进程，并且这些进程之间相互通信，那么即使在支持其他应用程序(如Mesos/YARN)的集群管理器上运行它也相对容易。

- 驱动程序必须在其整个生命周期中侦听并接受来自执行器的传入连接(例如，参见spark.driver)。端口在网络配置部分)。因此，驱动程序必须是可从工作节点进行网络寻址的。

- 因为驱动程序在集群上调度任务，它应该在靠近工作节点的地方运行，最好是在相同的局域网上。如果您想远程向集群发送请求，最好是向驱动程序打开一个RPC，让它在附近提交操作，而不是在远离工作节点的地方运行一个驱动程序。

> 原文：https://spark.apache.org/docs/latest/cluster-overview.html




### Spark V.S MapReduce

| MapReduce                                                    | Spark                                                        |
| :----------------------------------------------------------- | ------------------------------------------------------------ |
| 细粒度的资源申请<br />没执行一个task申请一个JVM进程，执行完就释放 | 粗粒度的资源申请<br />提前把executors资源创建好，然后以线程来执行task，<br />可以执行多个task |
| 中间结果写入磁盘                                             | 中间结果写入内存                                             |
|                                                              |                                                              |



### 数据读取

- 可以读取本地的数据
- 可以读取hdfs的数据([hdfs://mycluster/sparktest](hdfs://mycluster/sparktest))


### 集群资源类型

- Standalone

Spark自己的集群资源管理，由Master和Workers组成，很明显master有单点风险，Spark可以基于<font color='red'>Zookeeper</font>支持HA，

- Yarn



### 提交任务

spark-submit脚本提交jar包

### History

计算层把计算的日志存到hdfs，spark-history-server负责查看日志





### 疑问？

Worker和Executor的关系？

cores？

