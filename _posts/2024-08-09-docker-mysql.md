---
title: docker 安装 mysql5.7, redis7.0
description:  docker 安装 mysql5.7, redis7.0
categories: [docker, docekr 安装软件]
tags: [docker]
author: [xiao_e]
---


## mysql5.7

参考文章： https://cloud.tencent.com/developer/article/2356690

```shell
mkdir -p /Users/wy/dev_env/mysql_data/log
mkdir -p /Users/wy/dev_env/mysql_data/data
mkdir -p /Users/wy/dev_env/mysql_data/conf

docker run -p 3306:3306 --name mysql5.7_v2 \
-v /Users/wy/dev_env/mysql_data/log:/var/log/mysql \
-v /Users/wy/dev_env/mysql_data/data:/var/lib/mysql \
-v /Users/wy/dev_env/mysql_data/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:5.7

```

## Redis7.0
```shell
docker run -p 6379:6379 --name redis_7.0.10 -v /Users/wy/dev_env/redis_data/conf/redis.conf:/etc/redis/redis.conf  -v /Users/wy/dev_env/redis_data/data:/data -d redis:7.0.10 redis-server /etc/redis/redis.conf --appendonly yes

```