---
layout: post
title: ClickHouse表引擎TinyLog和Memoey
categories: [ClickHouse]
description: ClickHouse
keywords: ClickHouse
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## TinyLog

以列文件的形式保存在磁盘上，不支持索引，没有并发控制。一般保存少量数据的小表， 生产环境上作用有限。可以用于平时练习测试用。

```
create table t_tinylog ( id String, name String) engine=TinyLog;
```

## Memory

内存引擎，数据以未压缩的原始形式直接保存在内存当中，服务器重启数据就会消失。 读写操作不会相互阻塞，不支持索引。简单查询下有非常非常高的性能表现（超过 10G/s）。

一般用到它的地方不多，除了用来测试，就是在需要非常高的性能，同时数据量又不太 大（上限大概 1 亿行）的场景。
