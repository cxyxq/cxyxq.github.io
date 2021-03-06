---
layout: post
title: Java中的四种引用
categories: [jvm]
description: Java中强、软、弱、虚引用的区别
keywords: Jim
typora-root-url: ../../
---

Java中存在四种引用类型，强度由强到弱，下面分别来介绍

### 强引用

强引用在java程序中普遍存在，类似`Object obj = new Object();` 这类的引用，**只要强引用还在，垃圾收集器永远不会回收掉被引用的对象**，即使发生OOM，也不会。

### 软引用

软引用描述一些还有用但非必需的对象。对于软引用关联着的对象，在系统即**将要发生内存溢出异常**之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还是没有足够的内存，才会抛出内存溢出异常。Java中用`SoftReference`类来实现软引用。

### 弱引用

弱引用来描述非必需对象的，被弱引用关联的对象只能生存到下一次垃圾收集器发生之前。当垃圾收集器工作时，**无论当前内存是否足够，都会回收掉只被弱引用关联的对象**。Java中提供了`WeakReference`类来实现弱引用。

### 虚引用

虚引用被称做幽灵引用或者幻影引用，它是最弱的一种引用。 一个对象是否有虚引用的存在，不会对其生存时间构成影响，也无法通过虚引用来取得一个对象的实例。为对象设置虚引用的唯一目的时在这个**对象被收集器回收时收到一个系统通知**。Java中提供了`PhantomReference`类来实现虚引用。



参考：

<深入理解Java虚拟机 第2版>-3.2.3 再谈引用