---
layout: post
title: Netty常用组件总结
categories: [NIO]
description: Netty常用组件总结
keywords: NIO
typora-root-url: ../../
---


## Netty常用组件总结

### 线程模型

#### EventLoopGroup

#### ChannelHandler

用来处理IO事件或者拦截IO操作。

#### ChannelPipeline

内部维护1个`ChannelHandler`集合，用来处理或者拦截inbound,outbound事件。每个channel有自己的pipeline，它在channel被创建的时候自动创建(ChannelPipeline).

#### Unpooled

#### ChannelGroup

#### IdleStateHandler-心跳

#### WebSocket长连接

#### 编码、解码 - codec

##### Google Protobuf

#### 粘包，拆包

