---
layout: post
title: ClickHouse简介及特点
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

## ClickHouse简介及特点

### 简介

ClickHouse 是俄罗斯的 Yandex 于 2016 年开源的列式存储数据库（DBMS），使用 C++ 语言编写，主要用于在线分析处理查询（OLAP），能够使用 SQL 查询实时生成分析数据报告。

### 特点

#### 列式存储

行式存储：好处是想查某个人所有的属性时，可以通过一次磁盘查找加顺序读取就可以。但是当想 查所有人的年龄时，需要不停的查找，或者全表扫描才行，遍历的很多数据都是不需要的。

列式存储：想查所有人的年龄只需把年龄那一列拿出来即可。

列式存储的好处：

1.对于列的聚合，计数，求和等统计操作优于行式存储。

2.由于某一列的数据类型都是相同的，针对于数据存储更容易进行数据压缩，每一列选择更优的数据压缩算法，大大提高了数据的压缩比重。

3.由于数据压缩比更好，一方面节省了磁盘空间，另一方面对于cache也有了更大的发挥空间。

#### DBMS的功能

几乎覆盖了标准 SQL 的大部分语法，包括 DDL 和 DML，以及配套的各种函数，用户管理及权限管理，数据的备份与恢复。

#### 多样化引擎

ClickHouse 和 MySQL 类似，把表级的存储引擎插件化，根据表的不同需求可以设定不同 的存储引擎。目前包括合并树、日志、接口和其他四大类 20 多种引擎。

#### 高吞吐写入能力

ClickHouse 采用类 LSM Tree的结构，数据写入后定期在后台 Compaction。通过类 LSM tree 的结构，ClickHouse 在数据导入时全部是顺序 append 写，写入后数据段不可更改，在后台 compaction 时也是多个段 merge sort 后顺序写回磁盘。顺序写的特性，充分利用了磁盘的吞 吐能力，即便在 HDD 上也有着优异的写入性能。 

官方公开 benchmark 测试显示能够达到 50MB-200MB/s 的写入吞吐能力，按照每行 100Byte 估算，大约相当于 50W-200W 条/s 的写入速度。

#### 数据分区与线程级并行

ClickHouse 将数据划分为多个 partition，每个 partition 再进一步划分为多个 index  granularity(索引粒度)，然后通过多个 CPU核心分别处理其中的一部分来实现并行数据处理。 在这种设计下，单条 Query 就能利用整机所有 CPU。极致的并行处理能力，极大的降低了查 询延时。 

所以，ClickHouse 即使对于大量数据的查询也能够化整为零平行处理。但是有一个弊端 就是对于单条查询使用多 cpu，就不利于同时并发多条查询。所以对于高 qps 的查询业务， ClickHouse 并不是强项。

#### 性能对比

ClickHouse 像很多 OLAP 数据库一样，单表查询速度由于关联查询，而且 ClickHouse 的两者差距更为明显。
