---
layout: post
title: 零拷贝
categories: [Blog]
description: 零拷贝的原理分析
keywords: NIO
---

## 零拷贝
[零拷贝原理介绍](https://xunnanxu.github.io/2016/09/10/It-s-all-about-buffers-zero-copy-mmap-and-Java-NIO/ 零拷贝-mmap,nio等)

- 为什么内核内存空间有2次DMA拷贝？因为普通硬件DMA访问需要连续的内存空间
- 从操作系统角度来看，是零拷贝的 (因为没有数据从kernel space 拷贝到 user space)