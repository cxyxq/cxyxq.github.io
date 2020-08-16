---
layout: post
title: Mybatis缓存学习
categories: [jvm]
description: 关于Mybatis一级、二级缓存自己学习的一些总结
keywords: mybatis
typora-root-url: ../../
---



## 注册中心

### Eureka

### Zookeeper

### Consul

### Nacos

## 服务调用

### RestTemplate

### Ribbon

客户端负载均衡。

主启动类加

```java
@RibbonClient(value = "name",configuration = {MyRibbonConf.class})
```

```java
@Configuration
public class MyRibbonConf {
    @Bean
    public IRule iRule() {
        return new RandomRule();
    }
}
```



### OpenFeign

## 服务熔断

### Hystrix

## 网关

### Gateway

## 配置中心

### Config

### Bus

## 消息

### Stream

## 服务链路跟踪

### Sleuth



