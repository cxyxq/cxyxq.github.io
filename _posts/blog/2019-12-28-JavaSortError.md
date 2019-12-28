---
layout: post
title: Collections.sort排序异常,Comparison method violates its general contract!
categories: [Java异常]
description: Collections.sort排序异常
keywords: 排序, 异常
---

## java.lang.IllegalArgumentException: Comparison method violates its general contract!



线上某功能反馈无法正常使用，查询日志得到这么一个错误:

```java
Caused by: java.lang.IllegalArgumentException: Comparison method violates its general contract!
  at java.util.TimSort.mergeLo(TimSort.java:773)
  at java.util.TimSort.mergeAt(TimSort.java:510)
  at java.util.TimSort.mergeCollapse(TimSort.java:435)
  at java.util.TimSort.sort(TimSort.java:241)
  at java.util.Arrays.sort(Arrays.java:1512)
  at java.util.ArrayList.sort(ArrayList.java:1454)
  at java.util.Collections.sort(Collections.java:175)
```



待补充.....